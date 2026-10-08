**HazeAIHub 智能客服架构演进**

从单体状态机到 ReAct Agent 的工程实践

作者：张林 \| 项目：Haze-AI-Hub \| 2026年08月02日

# 一、引言：一次"不得不做"的重构

HazeAIHub 是一个面向企业内部的智能客服平台，后端基于 Spring Boot 3.4 + Spring AI 1.1.2 构建，前端采用 React。系统按业务域划分为三大入口：IT 服务台（工单创建/查询/知识问答）、HR 服务（请假/打卡）、行政服务（设备申领/会议室预订）。核心能力是通过 LLM 驱动的多轮对话，帮助员工自助完成 IT 工单创建、知识检索、设备申领等操作。

项目第一版（以下简称"旧版本"）经过多轮迭代后，核心对话链路暴露出一系列结构性问题：一套从未真正跑通的 Skill 框架骨架代码、一个近 500 行的单体工单创建服务类、一份 if/else 堆叠的意图识别实现，以及一套隐藏在方法名里的隐式状态机。这些问题导致新增功能越来越困难、Bug 定位越来越耗时。

2026 年 7 月，我决定对 IT 服务核心链路进行一次彻底的架构重写：引入 ReAct Agent 状态机替代旧的单体状态机，用策略链重构意图识别模块，同时清理掉所有已废弃的 Skill 框架代码。这篇文章是对这次重构的完整记录。

# 二、旧架构解剖：我们到底在还什么技术债

## 2.1 Skill 框架：设计很好，从未跑通

旧版本中存在一个完整的 skill/ 包，包含 SkillExecutor 接口、SkillRouter 路由接口、IntentDetector 意图检测接口，以及四个 Skill 实现类（KnowledgeQASkill、TicketCreateSkill、TicketQuerySkill、TicketUrgentSkill）。架构设计思路清晰：意图检测 → 路由分发 → Skill 执行，是典型的策略模式 + 责任链的变体。

**然而，实际情况是：**

-   SkillRouter 和 IntentDetector 只有接口定义，没有实现类

-   TicketCreateSkill.execute() 的方法体是 return SkillResult.ok("工单创建待实现")——一个 TODO 占位符

-   其他三个 Skill 同样没有完整的业务逻辑

-   整个 skill/ 包没有任何调用方——ChatServiceImpl 走的是另一套 Handler 链路由

换句话说，这是一个设计完整但从未接入主链路的"空中楼阁"。它占用着 8 个 Java 源文件和一个专门的 result/SkillResult 类，却对系统行为没有任何实际贡献。

## 2.2 TicketCreateServiceImpl：500 行单体类的维护噩梦

旧版本的工单创建逻辑全部塞在 TicketCreateServiceImpl 这一个类里，实现了 ITicketCreateService 接口的 8 个方法：startCreate、continueCreate、abandonCreate、getState、hasOngoingState、startSuggest、continueSuggest。每次新增一个小需求（比如"建单前先检索知识库"），就往这个类里追加一个新的方法或分支。

**这个类的核心问题：**

-   状态管理隐式化：工单创建的三个阶段（SUGGESTING → COLLECT → CONFIRM）没有显式的状态类，全靠方法名和方法内的 if 分支区分

-   Redis 状态序列化脆弱：TicketCreateState 通过 JSON 序列化存入 Redis，字段变更时极易出现反序列化失败

-   知识检索和工单创建耦合：startSuggest 里同时做了知识库检索和状态初始化，改一个影响另一个

-   测试几乎不可能：没有状态接口，无法 mock 中间态，只能做端到端集成测试

更要命的是，"SUGGESTING" 态的逻辑本身存在设计缺陷——用户在建单前看到知识库建议回复后，如果直接描述问题细节（而非先确认"需要建单"），系统会陷入状态混乱。

## 2.3 IntentDetectionServiceImpl：if/else 堆叠

旧版本的意图识别实现 IntentDetectionServiceImpl 是一个典型的过程式代码块：先用 LLM（DashScopeChatModel）调用检测意图，失败了就 fallback 到数据库关键词匹配，还没有就查服务目录表——三层兜底逻辑写在一个方法里，每层的异常处理和日志混杂在一起。

**具体问题：**

-   LLM 调用没有超时控制：网络抖动时整个请求被阻塞

-   三层策略之间没有清晰的接口抽象，加第四种策略需要改主流程代码

-   filterByDomain 逻辑直接写在 service 里，HR/ADMIN 域的特殊处理散落在各处

## 2.4 隐式状态机：状态藏在方法名里

旧版本工单创建的三个核心方法——startSuggest、continueCreate、continueSuggest——实际上构成了一台状态机，但没有任何代码层面体现这一点。状态转换规则散落在各个方法内部的 if/else 中：

// TicketCreateServiceImpl.continueCreate() 中的状态判断

if ("SUGGESTING".equals(state.getStatus())) {

return continueSuggest(userId, prompt, state, sessionId);

}

if ("CONFIRM".equals(state.getStatus())) {

// 处理确认/取消逻辑...

}

// COLLECT 状态...

要理解完整的对话流程，你需要在三个方法之间来回跳转，在脑海中拼出状态转换图。新增一个状态意味着修改至少两个方法的 if 条件，漏改一处就出 Bug。

# 三、新架构设计：ReAct Agent + 策略链

## 3.1 核心思想：从"硬编码流程"到"LLM 自主决策"

新架构的核心变化是：不再用 Java 代码硬编码"先问标题、再问描述、再问优先级"的对话流程，而是让 LLM 在每一轮根据当前状态和用户输入，自主决定下一步动作。这本质上是一个 ReAct （Reasoning + Acting）循环：LLM 思考 → 决定是否调用工具 → 观察工具结果 → 再次思考 → 最终回复用户。

同时，对话流程被建模为显式的三态状态机：DIAGNOSING（诊断） → COLLECTING（收集信息） → CONFIRMING（确认建单）。每个状态是一个独立的 DialogState 实现类，状态转换由 AgentStateMachine 的事件驱动机制统一管理。

## 3.2 ReAct 循环：Thought → Action → Observation → Respond

AgentStateMachine 是整个新架构的引擎核心，位于 ai/agent/core/AgentStateMachine.java。它的 process() 方法对每个状态执行 handle()，根据返回的 AgentEvent 类型分派：

| **事件类型**     | **处理逻辑**                                                                      | **说明**             |
|------------------|-----------------------------------------------------------------------------------|----------------------|
| RESPOND          | 直接 emit 给 SSE 流                                                               | 正常的 LLM 回复内容  |
| STATE_TRANSITION | 更新 ctx.state.status，等待下一轮用户输入                                         | 状态推进但不自动递归 |
| COMPLETE         | 标记 ctx.finished = true，清理 Redis 状态                                         | 对话正常结束         |
| TOOL_CALL        | 在 boundedElastic 线程池执行工具 → 将结果注入 ctx.lastToolResult → 递归回当前状态 | ReAct 自循环核心     |

关键设计点：TOOL_CALL 事件会触发"递归回当前状态"——LLM 拿到工具执行结果后重新决策，可能再次调用工具，也可能直接回复用户。这就是 ReAct 的 Thought → Action → Observation 循环。而 STATE_TRANSITION 不会自动递归——它只是更新状态，等待用户的下一条消息再触发下一轮。

## 3.3 显式状态机：三个独立的状态类

**每个状态类职责单一、可独立测试：**

-   DiagnosingState：分析用户描述，提取关键信息（问题类型、紧急程度），判断是否可以进入收集阶段。如果信息不够，追问用户

-   CollectingState：逐字段收集工单信息（标题、描述、优先级、分类），用 LLM 判断当前已收集到哪些字段、还缺哪些。所有字段收集完毕后自动转换到 CONFIRMING

-   ConfirmingState：将收集到的工单信息汇总展示给用户确认。用户确认 → 调用 CreateTicketTool 创建工单 → COMPLETE。用户取消 → 清状态 → COMPLETE

状态类通过依赖注入获取 ToolRegistry（工具注册表），LLM 在需要时自主决定调用 knowledge_search（检索知识库）或 create_ticket（创建工单）工具，而不是由硬编码逻辑决定。

## 3.4 意图识别策略链：LLM → 关键词 → 服务目录 → 兜底

新的意图识别模块位于 ai/intent/ 包，使用策略模式重构。核心接口 IntentStrategy 只定义一个方法：Mono\<IntentDetectionResult\> detect(String userMessage, String domain)。

**四种策略按优先级依次尝试：**

| **优先级** | **策略**     | **实现类**            | **说明**                             |
|------------|--------------|-----------------------|--------------------------------------|
| 1          | LLM 语义识别 | LlmIntentStrategy     | 调用大模型分析意图，3 秒超时保护     |
| 2          | 关键词匹配   | KeywordIntentStrategy | 查数据库 intent_keyword 表，精确匹配 |
| 3          | 服务目录匹配 | CatalogIntentStrategy | 查 biz_service_catalog 表，模糊匹配  |
| 4          | 兜底         | （内置）              | 返回 text 意图，走通用对话           |

每条策略返回 Mono.empty() 表示"我没匹配到，换下一条"，用 switchIfEmpty 串联。新增策略只需实现 IntentStrategy 接口并加入链即可，完全符合开闭原则。LLM 策略设置了 3 秒超时，既保证语义识别的准确性，又防止网络抖动拖垮整个链路。

## 3.5 对话记忆：跨会话的 ConversationMemoryService

新架构引入 ConversationMemoryService，在用户开始建单流程时自动加载该会话的历史对话摘要，注入到 TicketCreateState.suggestionContent 字段中。这解决了旧版本的一个典型问题：用户在同一个会话中先进行了几轮知识问答，然后说"还是不行，帮我建个工单"，旧版本会丢掉前面的上下文，新版本则会让 Agent 带着历史摘要进入诊断状态。

# 四、关键改造实录

## 4.1 文件变更统计

| **变更类型** | **数量**     | **详情**                                                                                                                       |
|--------------|--------------|--------------------------------------------------------------------------------------------------------------------------------|
| 新增         | 21 个文件    | ai/agent/（16 个） + ai/intent/（5 个）                                                                                        |
| 删除         | 13 个文件    | skill/（8 个） + TicketCreateServiceImpl + ITicketCreateService + IntentDetectionServiceImpl + SkillResult + cscontroller.java |
| 修改         | 3 个文件     | ChatServiceImpl（核心分流逻辑）、ChatPipeline（think 标签处理修复）、TicketCreateHandler（接入 Agent）                         |
| 不变         | \~240 个文件 | ai-common、ai-infrastructure、ai-domain 三模块以及大部分 Controller/Service/Config 完全不变                                    |

## 4.2 ChatServiceImpl 的分流改造

ChatServiceImpl.textChat() 是系统的总入口。新版本的改造重点在于 IT 域的分流逻辑，核心改动不到 30 行，但改变了整个请求的处理流程：

// ChatServiceImpl.textChat() — IT 分支的新分流逻辑

// 1. 如果用户已在 Agent 多轮对话中（DIAGNOSING/COLLECTING/CONFIRMING）

// 直接 continueDialog，跳过意图识别，不受新消息意图干扰

if (agentDialogService.hasOngoingDialog(userId)) {

return initialFlux.concatWith(

chatPipeline.wrap(ctx, agentDialogService.continueDialog(userId, prompt, finalSessionId))

);

}

// 2. 不在 Agent 流程中 → 意图识别

return intentDetectionService.analyzeIntentReactive(prompt, ctx.getDomain())

.flatMapMany(intent -\> {

// 3. ticket_create 意图 → 启动 Agent 对话

if ("ticket_create".equals(intent.getIntent())) {

return chatPipeline.wrap(ctx, agentDialogService.startDialog(...));

}

// 4. 其他意图 → 走原有 Handler 链（ticket_query / knowledge_qa / 兜底）

return chatPipeline.wrap(ctx, handlerRegistry.findFirstMatch(ctx).get().handle(ctx));

});

这个设计的精妙之处在于：Agent 接管和旧 Handler 链可以平滑共存。ticket_create 意图走新 Agent，ticket_query/ticket_urgent 等走旧的 TicketActionHandler，knowledge_qa 走旧的 KnowledgeQAHandler，DefaultChatHandler 兜底。新旧代码之间通过 HandlerRegistry 解耦，互不感知。

## 4.3 Agent 状态持久化：Redis + 30 分钟 TTL

AgentDialogService 将 TicketCreateState 序列化为 JSON 存入 Redis，key 为 agent_dialog:{userId}，TTL 设置为 30 分钟。每次用户发消息时，先从 Redis 恢复状态，执行状态机的 process()，处理完毕后再序列化回 Redis。对话正常结束时（COMPLETE 事件），主动删除 Redis key。

**这个设计保证了：**

-   服务重启不丢状态：状态在 Redis 里，不在 JVM 内存里

-   30 分钟无操作自动过期：防止 Redis 内存泄漏

-   会话隔离：state.sessionId 校验防止跨会话状态污染

## 4.4 ChatPipeline 的 Bug 修复

ChatPipeline 负责 SSE 流的"后置处理"：收集流式 chunk → 组装成完整回复 → 保存 AI 消息 → 更新会话时间 → 生成标题。旧版本中存在一个跨 chunk 的 think 标签截断问题：

// 旧版本：逐 chunk 提取 think 标签（有 Bug）

.doOnNext(chunk -\> {

if (chunk.contains("\<think\>") \|\| chunk.contains("\</think\>")) {

String extracted = extractThinkContent(chunk);

reasoningBuilder.append(extracted);

return; // ← 整个 chunk 被丢弃，不进入 content

}

if (!isSystemMarker(chunk)) {

contentBuilder.append(chunk);

}

})

当 \<think\> 和 \</think\> 跨两个 SSE chunk 时，第一个 chunk 末尾可能不是完整的闭合标签，导致标签中间的正常文本被错误丢弃。新版本改为：先收集全部 chunk，在 doOnComplete 中对完整文本统一执行 regex 清理，支持 \<think\>、\<thinking\>、\<thought\> 三种变体。彻底消除跨 chunk 问题。

# 五、踩坑与修复

## 5.1 P0：search_knowledge 分支缺失导致整个 RAG 链路失效

这是本次重构中发现的最严重问题。在排查 CollectingState 的逻辑时，发现 Agent 的 ToolRegistry 中注册了 KnowledgeSearchTool，但 DiagnosingState 的 LLM 决策 prompt 中并没有告知模型"你可以调用 search_knowledge 来检索知识库"——模型根本不知道该工具有什么用、什么时候用。这意味着我们花时间调优的 HyDE 检索 + 混合检索 + ReRank 重排序链路，实际上在 Agent 流程中从未被触发过。修复方式：在 DiagnosingState 和 CollectingState 的 system prompt 中明确列出可用工具及其用途，并在 few-shot 示例中展示何时应该调用 search_knowledge。

## 5.2 P0：用户说"谢谢"后状态不清理的内存泄漏

早期版本的 AgentDialogService.afterProcess() 中，只有当 ctx.isFinished() 为 true 时才清理 Redis 状态。但"对话结束"的判断完全依赖 LLM 是否返回 COMPLETE 事件。当用户说"谢谢"或"好的"时，LLM 可能只返回 RESPOND 而不返回 COMPLETE，导致 Redis key 永远不被清理——30 分钟 TTL 是兜底，但在高并发场景下会造成 Redis 内存持续增长。

修复方案：在 DiagnosingState 的 prompt 中增加明确的"对话结束判断规则"，当用户表达感谢、确认完成、或对话主题已经结束时，要求 LLM 返回 COMPLETE 事件。同时在 AgentDialogService 中增加兜底逻辑：如果连续两轮没有状态推进且用户输入很短（\<5字），主动清理状态。

## 5.3 P1：跨 chunk 的 think 标签截断

已在 4.4 节详述。修复方式：从"逐 chunk 正则提取"改为"doOnComplete 时对完整文本统一清理"。这个修复同时解决了三个问题：跨 chunk 截断、嵌套标签处理、以及 \<thinking\>/\<thought\> 变体支持。

## 5.4 P1：HttpClient 每次 new 导致的连接池泄漏

旧版本的 HttpClientUtil 在每次 HTTP 调用时都 new 一个 CloseableHttpClient，用完后虽然调用了 close()，但没有任何连接池复用机制。在高并发文件上传场景下，频繁创建/销毁连接导致端口耗尽和 TIME_WAIT 堆积。修复：改用单例的 PoolingHttpClientConnectionManager，配置 maxTotal=50、defaultMaxPerRoute=10，并通过 @PreDestroy 在应用关闭时优雅释放。

## 5.5 日期计算的"工程正确"与"数学正确"

旧版本 ChatServiceImpl.handleDomainChat() 中计算"下周一"的逻辑：

// 旧版本：数学正确，但工程错误

LocalDate nextMonday = today.plusDays(7 - now.getDayOfWeek().getValue() + 1);

这行代码在周日（getValue()=7）时计算出的 7-7+1=1 天后的日期其实是周一但跨了一周，逻辑上是对的但可读性极差。新版本改用标准 API：

// 新版本：语义清晰

LocalDate nextMonday = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));

这个修改不改变行为，但让代码意图一目了然。类似的"看起来一样但其实更正确"的修改还有多处，比如 AdminManageController 中处理前端提交的日期格式时，之前靠 DateTimeFormatter 硬编码 yyyy-MM-dd，新版本改为接受 ISO 标准格式并使用 LocalDate.parse() 自动适配。

# 六、成果与数据

**代码指标变化：**

| **指标**            | **旧版本**                       | **新版本**                   | **变化**                       |
|---------------------|----------------------------------|------------------------------|--------------------------------|
| IT 域核心链路源文件 | \~15 个                          | \~25 个（含 agent + intent） | 新增 10 个，但职责更清晰       |
| 最大单类行数        | TicketCreateServiceImpl \~500 行 | 单个状态类 \<200 行          | 单一职责，可读性显著提升       |
| 意图识别策略        | 1 个类，3 层嵌套 if              | 1 接口 + 4 策略类            | 开闭原则，新增策略无需改主流程 |
| Skill 框架代码      | 8 个文件（全部未使用）           | 0 个                         | 彻底清理                       |
| 工单状态建模        | 隐式（方法名暗示）               | 显式（3 个独立 State 类）    | 状态转换可追溯、可单测         |
| Agent 架构          | 无                               | ReAct 循环 + 事件驱动        | LLM 自主决策替代硬编码流程     |

**架构层面收获：**

-   状态显式化：DIAGNOSING → COLLECTING → CONFIRMING 的状态转换在代码中清晰可见，每个状态的 prompt、工具、退出条件独立管理

-   事件可观测：AgentEvent（RESPOND/STATE_TRANSITION/COMPLETE/TOOL_CALL）让对话流程可监控、可调试

-   可扩展性：HR/ADMIN 域未来可以接入同一套 Agent 状态机；跨域转接的预留接口已在 ChatServiceImpl 中定义

-   测试友好：每个 DialogState 可以独立 mock ToolRegistry 进行单元测试，不需要启动完整 Spring 容器

# 七、反思：重构教会我们的事

## 7.1 "能跑"不等于"对"

旧版本的 TicketCreateServiceImpl 是能正常创建工单的，Skill 框架也不会报错（因为根本没被调用）。但这种"能跑"掩盖了架构层面的债务：隐式状态机、未使用的框架代码、脆弱的知识检索耦合。如果只看功能测试通过就认为"没问题"，这些债务会越积越多，直到某天一个小需求改动引发连锁崩溃。

## 7.2 Prompt 工程需要兜底，但兜底不能替代架构设计

新架构大量依赖 LLM 的 prompt 指令来控制 Agent 行为（状态判断、工具选择、字段收集）。这带来了极大的灵活性，但也引入了一个新风险：prompt 的微小调整可能产生意料之外的行为变化。因此，意图识别层保留了非 LLM 的兜底策略（关键词 + 服务目录），Agent 层也有 TTL 超时和脏状态清理的兜底逻辑。关键原则是：LLM 负责"智能"，代码负责"安全"。

## 7.3 流式架构里，阻塞调用是原罪

整个系统基于 Spring WebFlux 的 SSE 流式响应构建。旧版本的 IntentDetectionServiceImpl 中，LLM 调用没有超时设置，一旦 DashScope API 响应慢，整个请求线程被阻塞。新版本中所有 LLM 调用都设置了超时（意图识别 3 秒、Agent 决策 10 秒），工具调用在 boundedElastic 线程池执行，确保不阻塞事件循环。在响应式架构中，每增加一个 .block() 调用都要三思——它是不是真的必须同步等待。

## 7.4 删除代码和写代码同样重要

这次重构删除了 13 个文件、近 1000 行代码。Skill 框架的清理尤其解气——一套设计优雅但从未工作的代码，终于不再占据包结构和新人理解的心智负担。重构不应该是"一边加新代码，一边留旧代码"——旧的不去，新的不来。每一次提交都应该问自己：有没有因为这次改动而变得不再需要的代码？有的话，删掉它。

# 附录：完整文件变更清单

## 新增文件（21 个）

ai-server/.../ai/agent/

├── AgentDialogService.java \# Agent 编排服务（Redis 状态管理）

├── DiagnosingState.java \# 诊断态（分析用户描述）

├── CollectingState.java \# 收集态（逐字段收集工单信息）

├── ConfirmingState.java \# 确认态（汇总确认 + 建单）

├── ReActDecision.java \# LLM 决策模型

├── ConversationMemoryService.java \# 历史对话摘要记忆

├── core/AgentEvent.java \# 事件类型定义

├── core/AgentStateMachine.java \# ReAct 循环引擎

├── core/DialogContext.java \# 对话上下文

├── core/DialogState.java \# 状态抽象接口

├── rag/HydeSearchService.java \# HyDE 检索服务

├── tool/AgentTool.java \# 工具抽象

├── tool/CreateTicketTool.java \# 创建工单工具

├── tool/KnowledgeSearchTool.java \# 知识检索工具

├── tool/ToolInitializer.java \# 工具注册初始化

└── tool/ToolRegistry.java \# 工具注册表

ai-server/.../ai/intent/

├── IntentDetectionService.java \# 新意图识别实现（策略链）

└── strategy/

├── IntentStrategy.java \# 策略接口

├── LlmIntentStrategy.java \# LLM 语义策略

├── KeywordIntentStrategy.java \# 关键词匹配策略

└── CatalogIntentStrategy.java \# 服务目录策略

## 删除文件（13 个）

ai-server/.../skill/

├── core/SkillContext.java \# 未使用的 Skill 上下文

├── core/SkillExecutor.java \# 未使用的 Skill 接口

├── impl/KnowledgeQASkill.java \# TODO 占位实现

├── impl/TicketCreateSkill.java \# TODO 占位实现

├── impl/TicketQuerySkill.java \# 空壳实现

├── impl/TicketUrgentSkill.java \# 空壳实现

├── router/IntentDetector.java \# 未实现的接口

└── router/SkillRouter.java \# 未实现的接口

ai-server/.../

├── result/SkillResult.java \# Skill 结果类

├── ai/it/service/ITicketCreateService.java \# 旧工单创建接口

├── ai/it/service/impl/TicketCreateServiceImpl.java \# 旧工单创建实现（\~500行）

├── ai/service/impl/IntentDetectionServiceImpl.java \# 旧意图识别实现

└── cscontroller.java \# 拼写错误的文件
