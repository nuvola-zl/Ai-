# 从文件上传到精准回答：企业级 RAG 知识库的工程化实践

> 背景：Haze AI Hub，基于 Spring Boot 3.2 + Spring AI + PostgreSQL(pgvector) + Redis 构建的企业内部 AI 知识库系统，支撑 IT 运维助手的知识检索与问答。

## 一、RAG 的工程现实：不是调通 API 就算完工

在构建企业内部 AI 助手的过程中，我们很快意识到一个事实：**RAG（检索增强生成）的质量瓶颈，80% 不在 LLM，而在检索链路。**

当用户询问”VPN 连接不上怎么办”时，系统需要在一秒内完成：文件解析 → 文本分片 → 向量入库 → Query 改写 → 混合检索 → 上下文扩展 → 重排序 → Prompt 组装 → 流式生成。任何一个环节的工程缺陷，都会直接体现在最终回答的准确性上。

本文从实际落地出发，梳理知识库全链路中的关键工程决策，以及面向生产环境的专业化演进方向。

## 二、文件上传与去重：从哈希校验到语义感知

### 2.1 项目初期的做法

文件上传层采用常规的三层校验：MIME 类型白名单、100MB 大小限制、SHA256 哈希去重。

`String`` sha256 ``=`` Sha256Util``.``calculate``(``file``.``getInputStream``());`  
`Optional``<``KbMedia``>`` existing ``=`` mediaMapper``.``findByLibraryIdAndSha256``(``libraryId``,`` sha256``);`  
`if`` ``(``existing``.``isPresent``())`` ``{`  
`    ``return`` ``toResponse``(``existing``.``get``());`` ``// 相同文件直接拒绝`  
`}`

这套机制能防住”同一文件重复上传”，但存在明显的边界盲区：用户修改了一个字后重新上传，SHA256 完全不同，系统将其视为全新文件，导致知识库中出现内容高度相似的多份文档。

### 2.2 专业做法：双层去重与版本管理

生产级知识库的去重策略通常分为两层：

**第一层：文件级硬去重** - 仍以 SHA256 为基准，防止完全重复的文件占用存储与向量索引 - 同时维护 `(library_id, sha256)` 唯一索引，数据库层兜底

**第二层：内容级软去重** - 文件解析完成后，取前 3 个 chunk 生成向量，与已有文档的对应 chunk 做相似度比对 - 若 Top-3 相似度均大于 0.95，触发”覆盖更新”流程：先删除旧 Media 下的所有 chunks，再插入新版 - 若相似度在 0.80\~0.95 之间，标记为”疑似重复”，进入人工审核队列

更进一步，应引入**版本管理**：`KbMedia` 表增加 `version` 字段，旧版本不物理删除，而是将关联 chunks 的 `status` 置为 `archived`。这样既能追溯历史，又避免向量空间被旧版本污染。

## 三、文本分片：粗粒度与细粒度的工程权衡

### 3.1 项目中的分片策略

系统采用”段落级分片”为主、“固定长度硬切”为辅的混合策略：

`// 段落分割（兼容多空行）`  
`String``[]`` paragraphs ``=`` text``.``split``(``"``\n``\s``*``\n``"``);`  
  
`// 单段落超过 1200 tokens 时，按句子边界进一步拆分`  
`if`` ``(``paraTokenSize ``>`` maxParagraphSize``)`` ``{`  
`    chunks``.``addAll``(``splitBySentences``(``paragraph``));`  
`}`  
  
`// 代码/表格类内容，按固定字符长度切分，不在代码内部找句子边界`  
`if`` ``(``isCodeLike``(``paragraph``))`` ``{`  
`    chunks``.``addAll``(``splitByFixedLength``(``paragraph``));`  
`}`

同时配置了 20% 的重叠率，通过从后往前找句子边界提取重叠区，保证分片衔接处的语义完整性。

### 3.2 按页解析的隐性假设与局限

PDF 解析采用 PDFBox 按页抽取：`stripper.setStartPage(pageNum); stripper.setEndPage(pageNum);`，然后对每页文本独立分片。

这种设计隐含了一个假设：**一页只承载一个主题。** 但在实际 SOP 文档中，“电脑蓝屏处理”和”电脑无法开机处理”可能写在同一页。按页解析后，两个主题被送入同一个分片池，`ChunkingService` 虽然按段落切分，但如果原文排版紧凑、段落间距不足，仍可能出现跨主题污染。

### 3.3 专业做法：语义分片与层次化切割

业界更成熟的方案是**语义分片（Semantic Chunking）**：

-   **标题驱动**：在解析阶段识别文档的标题层级（如”一、故障现象”“二、排查步骤”），以标题为边界切分 chunk，确保每个 chunk 内部主题单一
-   **滑动窗口兜底**：对于无明确标题的长段落，使用固定长度滑动窗口（如 512 tokens）+ 重叠区（128 tokens）兜底
-   **元信息注入**：在 chunk metadata 中记录 `heading_level`、`parent_section`，检索时可根据标题层级做过滤或加权

对于 PDF 这类版式文档，可结合 `PDFTextStripperByArea` 或商用解析服务（如 Azure Document Intelligence）提取标题结构，而非单纯按页抽取。

## 四、向量入库：Embedding 之外的关键细节

### 4.1 项目中的入库流程

解析后的文本通过 Spring AI 的 `Document` 抽象批量入库：

`Document`` doc ``=`` ``Document``.``builder``()`  
`    ``.``text``(``content``)`  
`    ``.``metadata``(``Map``.``of``(`  
`        ``"page"``,`` pageNum``,`  
`        ``"fileName"``,`` media``.``getFileName``(),`  
`        ``"libraryId"``,`` libraryId``,`  
`        ``"mediaId"``,`` mediaId``,`  
`        ``"chunkIndex"``,`` chunkIndex``++`  
`    ``))`  
`    ``.``build``();`  
  
`vectorStore``.``add``(``documents``);`` ``// 内部调用 EmbeddingModel 生成向量并写入 pgvector`

### 4.2 专业做法：批量控制、事务隔离与元数据规范

**批量写入控制**：Embedding API 通常有 RPM 和 TPM 限制。应将 chunks 分批（如每批 20\~50 条），批次间加入指数退避重试，避免触发限流。

**事务边界**：解析 → 分片 → 生成 embedding → 写入数据库，整个链路不应放在一个长事务中。推荐的做法是： 1. 先异步完成”解析 + 分片 + embedding 生成” 2. 再开启一个短事务，批量执行 `INSERT INTO kb_chunk` 3. 最后更新 `kb_media.status = 'PARSED'`

**元数据规范**：metadata 是 RAG 系统的”隐形索引”。建议强制包含以下字段： - `source_type`: `OFFICIAL` / `FEEDBACK` / `CANDIDATE`，用于检索时的来源隔离 - `file_type`: `PDF` / `WORD` / `MD`，不同文件类型的可信度不同 - `created_at`: 用于时间衰减排序，旧文档降权 - `page` / `chunk_index`: 用于上下文扩展和溯源

## 五、混合检索：单一召回策略的天花板

### 5.1 项目中的检索架构

系统采用 PostgreSQL + pgvector，在数据库层单 SQL 完成向量检索 + 全文检索 + RRF 融合：

`WITH`` vector_search ``AS`` (`  
`    ``SELECT`` ``id``, ``ROW_NUMBER``() ``OVER`` (``ORDER`` ``BY`` embedding ``<=>`` ?:``:vector``) ``AS`` ``rank``,`  
`           ``1`` ``-`` (embedding ``<=>`` ?:``:vector``) ``AS`` vector_score`  
`    ``FROM`` kb_chunk ``WHERE`` library_id ``=`` ? ``ORDER`` ``BY`` embedding ``<=>`` ?:``:vector`` ``LIMIT`` ?`  
`),`  
`fulltext_search ``AS`` (`  
`    ``SELECT`` ``id``, ``ROW_NUMBER``() ``OVER`` (``ORDER`` ``BY`` ts_rank_cd(``..``.) ``DESC``) ``AS`` ``rank``,`  
`           ts_rank_cd(``..``.) ``AS`` bm25_score`  
`    ``FROM`` kb_chunk ``WHERE`` library_id ``=`` ? ``AND`` to_tsvector(``'simple'``, content) @@ ``query`  
`    ``ORDER`` ``BY`` bm25_score ``DESC`` ``LIMIT`` ?`  
`)`  
`SELECT`` ``COALESCE``(v.``id``, f.``id``) ``AS`` ``id``,`  
`       (``COALESCE``(rrf_score(v.``rank``, ``60``), ``0``) ``+`` ``COALESCE``(rrf_score(f.``rank``, ``60``), ``0``)):``:float`` ``AS`` rrf_score`  
`FROM`` vector_search v ``FULL`` ``OUTER`` ``JOIN`` fulltext_search f ``ON`` v.``id`` ``=`` f.``id`  
`ORDER`` ``BY`` rrf_score ``DESC`` ``LIMIT`` ?`

### 5.2 为什么需要混合检索？

-   **向量检索**擅长语义匹配：用户问”连不上网”能召回”网络连接失败”，但对专有名词（如”ERR-2001”）或精确短语容易漏召
-   **BM25 全文检索**擅长关键词匹配：对错误码、型号、命令行等精确文本召回率高，但无法理解同义词
-   **RRF（Reciprocal Rank Fusion）**将两路结果按排名融合，不依赖绝对分数，避免了向量分数与 BM25 分数量纲不一致的问题

### 5.3 专业做法：Query 改写与检索参数调优

**Query 改写（Query Rewriting）**：用户原始 query 往往是口语化的，直接用于向量检索效果不佳。应在检索前增加改写层： - 口语 → 书面语：`vpn连不上` → `VPN 连接失败 故障排查` - 指代消解：结合对话历史，将”这个”“它”替换为具体实体 - 关键词抽取：提取核心实体（如”Windows”“认证失败”），用于 BM25 的精确匹配

**参数调优**： - `ef_search`（HNSW）：向量检索的探针数，值越大召回率越高但速度越慢。建议根据数据量动态调整：10 万条以下用 100，百万级用 200+ - `rrf_k`：RRF 平滑因子，通常固定为 60。若某一路检索质量明显偏低，可通过调整 `vectorTopK` 与 `bm25TopK` 的召回比例来平衡 - `iterative_scan = relaxed_order`：pgvector 的 HNSW 优化参数，允许在构建阶段放宽顺序约束，提升高并发下的检索稳定性

## 六、命中后处理：上下文扩展与重排序

### 6.1 项目中的上下文扩展

向量检索召回的是单个 chunk，但 chunk 可能处于文档中段，前后文丢失。系统实现了 `expandWithContext`：

1.  按 `mediaId` 对命中结果分组
2.  每组内按 `chunkIndex` 排序，合并连续区间（如 5,6,7 → \[5,7\]）
3.  对连续区间查前 1 个 + 后 1 个邻居，与区间内 chunk 合并去重
4.  按 `chunkIndex` 顺序拼接成”超级 chunk”

`// 伪代码核心逻辑`  
`Map``<``Integer``,`` ``String``>`` indexToContent ``=`` ``new`` ``TreeMap``<>();`  
`chunkMapper``.``findNeighbors``(``mediaId``,`` rangeStart``,`` ``1``).``forEach``(``...``);`` ``// 前邻居`  
`chunkMapper``.``findNeighbors``(``mediaId``,`` rangeEnd``,`` ``1``).``forEach``(``...``);``   ``// 后邻居`

### 6.2 专业做法：滑动窗口与 Cross-Encoder 重排序

**上下文扩展的演进**： - 固定窗口（前后各 1 个）在简单文档中有效，但对于”第 3 节引用第 1 节定义”的跨章节引用场景会失效 - 更专业的做法是基于**文档结构树**扩展：先解析出文档的章节层级，命中 chunk 后，向上回溯到父章节，将同级子章节一并纳入上下文

**重排序（ReRank）**： - 第一阶段的向量检索和 BM25 属于”粗排”，目标是高召回率 - 第二阶段使用 Cross-Encoder（如 `qwen3-rerank`）对粗排结果做精排，计算 query 与每个 chunk 的细粒度交互分数 - 实践中，粗排召回 Top 20\~50，精排取 Top 5\~10，能在延迟与精度间取得平衡

**性能优化**：ReRank 服务通常通过 HTTP 调用外部 API，需注意连接池复用与超时控制：

`private`` ``static`` ``final`` HttpClient HTTP_CLIENT ``=`` HttpClient``.``newBuilder``()`  
`    ``.``connectTimeout``(``Duration``.``ofSeconds``(``10``))`  
`    ``.``build``();`

避免在请求链路中重复创建 `HttpClient`，这是高并发下的常见资源泄漏点。

## 七、Prompt 工程：检索结果的最后一公里

### 7.1 项目中的分层 Prompt

系统区分了”官方文档”与”历史工单反馈”两类来源，在 Prompt 中做物理隔离：

`===== 官方文档（权威依据） =====`  
`[VPN配置指南-第2页] 1. 确认电脑已连接到公司内网...`  
  
`===== 历史工单反馈（仅供参考） =====`  
`[反馈] VPN连接失败排查-20240801 认证失败时检查系统时间...`

同时通过 `style` 参数支持两种回答模式： - `detail`：详细文档风格，引用来源，分点说明 - `concise`：精简客服风格，50 字以内，禁止复述原文

### 7.2 专业做法：来源置信度与未知内容拒绝

**来源置信度标注**：在 Prompt 中明确告知 LLM 不同来源的可信度差异： - 官方 SOP：置信度 1.0，可直接引用 - 工单反馈：置信度 0.6，需标注”根据历史工单…” - 外部网页：置信度 0.4，需交叉验证

**未知内容拒绝**：在 Prompt 中强制要求： \> “如果参考内容无法回答用户问题，明确说明’根据现有资料无法确定’，禁止编造参考内容中没有的信息。”

这能有效抑制 LLM 的幻觉，尤其在企业内网场景下，“不知道”比”胡说”更安全。

**Token 预算管理**：检索结果拼接后的 Prompt 长度需严格控制。建议在 `buildPrompt` 阶段增加截断逻辑：当上下文总长度超过模型上下文窗口的 70% 时，优先保留官方文档，截断反馈内容。

## 八、架构反思：尚未解决的专业化课题

在落地过程中，我们识别出以下尚未完全解决的工程课题，也是知识库系统从”可用”走向”好用”的必经之路：

### 8.1 语义去重

当前 SHA256 只能防完全重复，无法识别”改了一个字的新版”。下一步应引入向量级去重：新文档解析完成后，取其 Top-3 chunk 向量与库内已有文档做相似度扫描，相似度超过阈值时触发”覆盖更新”或”版本分支”。

### 8.2 跨主题分片

按页解析 + 段落分片的组合，在”一页多主题”的文档中仍存在跨主题污染。根治方案需要前置到解析层：引入标题层级识别，按语义边界切分，而非按物理页码切分。

### 8.3 知识库生命周期管理

一个健康的知识库需要”生老病死”： - **生**：新文档入库，标记 `status = current` - **长**：根据搜索点击率、采纳率动态调整 chunk 权重 - **老**：长期未访问（如 180 天）的 chunk，权重衰减或标记 `stale` - **死**：明确过时的知识，人工审核后标记 `archived`，不参与检索

### 8.4 官方库与反馈库的隔离架构

管理员上传的权威文档（Official KB）与工单自动沉淀的反馈知识（Feedback KB）应物理隔离： - 官方库：检索优先，神圣不可侵犯 - 反馈库：独立存储，仅在高置信度或人工审核后，以”补充方案”形式附加到官方库 - 检索时默认只查官方库，高级模式才联合查询反馈库，并在 Prompt 中明确标注来源差异

## 结语

RAG 系统的工程化，本质上是**在召回率、精确率、延迟、成本之间做持续权衡**。从文件上传到最终回答，每个环节都有”能跑”和”跑得好”之间的巨大鸿沟。

我们在 Haze AI Hub 项目中的实践验证了一条可行路径：SHA256 去重 → 段落级分片 → pgvector 混合检索 → 上下文扩展 → Cross-Encoder 重排序 → 分层 Prompt。但距离生产级的知识库系统，仍需在语义去重、跨主题分片、知识生命周期管理三个方向上持续迭代。

好的 RAG 系统，让用户感受不到检索的存在；坏的 RAG 系统，让用户在每一次问答中都能感受到知识的混乱。

*基于 Haze AI Hub 项目工程实践整理，技术栈：Spring Boot 3.2 + Spring AI + PostgreSQL(pgvector) + DashScope + PDFBox。*
