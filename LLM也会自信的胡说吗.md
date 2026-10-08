LLM 也会"自信地胡说"：当 AI 助手开始编造设备 ID

项目背景：Haze-AI-Hub 设备申领系统，接入 Spring AI 的 ChatClient + Function Calling，用户可以通过自然语言对话申请设备。联调测试时发现，AI 回复里的"设备 ID"和数据库完全对不上。

一、现象：AI 说的设备 ID，数据库里根本不存在

测试场景一（有库存申领）时，AI 的回复看起来很专业：

plain

您的 MacBook Pro 申领单 DEV-20260806-002 已成功提交！

当前流程：✅ 库存充足→ 正在配置设备（系统安装+权限设置）

⏳ 预计1-3 分钟内完成，完成后将自动通知您领取。

处理步骤：分配设备（ID:15）→ 安装系统 → 配置权限 → 通知领取

但查数据库验证时，发现实际分配的设备 ID 是 12，不是 15。

当时以为是后端接口返回错了。但继续测：

表格

| 单号             | AI 展示的 ID | 数据库真实 ID |
|------------------|--------------|---------------|
| DEV-20260806-002 | 15           | 12            |
| DEV-20260806-006 | 16           | 13            |
| DEV-20260806-007 | 17           | 13            |

规律很明显：15 → 16 → 17 递增，但数据库里根本没有 15、16、17 这几台设备。

执行 SELECT \* FROM admin_device_detail WHERE id = 17，结果是 Empty set。

AI 在编造不存在的设备 ID。

<img src="media/image1.png" style="width:5.76042in;height:4.07083in" />

二、排查：后端真的没返回设备 ID

先看后端的 DeviceTools.requestDevice 方法，这是 AI 申领设备的入口：

java

@Tool(description = "申领办公设备...")public String requestDevice(

@ToolParam(description = "用户ID") Long userId,

@ToolParam(description = "设备类型") String deviceType,

@ToolParam(description = "申领原因") String reason) {

// 1. 查重

String dupMsg = duplicateChecker.checkDuplicateDeviceRequest(userId, deviceType);

if (dupMsg != null) return dupMsg;

// 2. 提交申领（此时 DAG 还没跑，不知道会分配到哪个设备）

AdminDeviceRequest request = deviceRequestService.submitRequest(userId, deviceType, reason);

// 3. 异步触发 DAG

executor.execute(() -\> deviceDagService.executeDeviceDag(request.getRequestNo()));

// 4. 返回文案（注意：这里没有设备 ID！）

if (request.getStockStatus() == 1) {

return String.format(

"✅ 申领单 %s 已提交，当前仓库有库存，系统正在自动处理：\\n" +

"分配设备 → 安装系统 → 配置权限 → 通知领取\\n" +

"⏱️ 预计 1-3 分钟内完成...",

request.getRequestNo()

);

}

// ...}

关键发现：后端返回给 LLM 的文案是 "分配设备 → 安装系统..."，全文没有任何设备 ID。

而且此时 DAG 是异步执行的，提交瞬间系统根本不知道会分配到哪个设备。AI 展示的 ID:15 完全是它自己"创作"的。

三、根因：LLM Hallucination（幻觉）

这是大语言模型的经典问题——幻觉（Hallucination）。

LLM 的本质是概率生成模型，它不是数据库查询引擎。当它看到工具返回的文案 "分配设备 → 安装系统..." 时，它的"理解"是：

"这是一个步骤列表，但看起来不太完整。按照我训练数据里的常见模式，分配设备时通常会有一个设备 ID。用户可能想知道具体是哪台设备，我应该补一个合理的数字。"

然后它就在对话上下文中找线索，看到之前查进度时返回过 设备ID:12，于是"推断"下一次应该是 15、16、17...

为什么 System Prompt 没拦住？

当时的 System Prompt 是这样的：

java

public static final String ADMIN_SYSTEM_PROMPT = """

你是 HazeAIHub 智能行政助手...

规则：

\- 预定会议室必须确认：日期、时间段、人数/用途

\- 申领设备时，有库存直接分配，无库存自动发起采购

\- 用中文回复，语气友好专业

\- 【严禁编造时间】工具返回的文案已包含准确时间预期，直接原样回复，

不要自行添加"24小时"等描述

\- 【查询进度必须调用工具】禁止凭记忆回答设备状态

""";

问题很明显：只禁了"编造时间"，没禁"编造设备 ID"。LLM 很"听话"地不编时间了，但在其他方面继续自由发挥。

四、修复：给 LLM 加"护栏"

第一步：System Prompt 补规则

最直接有效的方式，在 System Prompt 里明确禁止：

java

public static final String ADMIN_SYSTEM_PROMPT = """

你是 HazeAIHub 智能行政助手...

规则：

\- 预定会议室必须确认：日期、时间段、人数/用途

\- 申领设备时，有库存直接分配，无库存自动发起采购

\- 用中文回复，语气友好专业

\- 【严禁编造时间】工具返回的文案已包含准确时间预期，直接原样回复

\- 【严禁编造设备信息】工具返回的文案已包含准确信息，直接原样回复。

不要自行添加设备ID、序列号、采购单号等编号，即使你认为"合理"也不行。

如果工具没有返回某个字段，直接说"未获取到"，不要脑补。

\- 【查询进度必须调用工具】禁止凭记忆回答设备状态

""";

第二步：后端在 DAG 完成后再查真实设备 ID

如果业务上确实需要在提交时就展示设备 ID，可以等 DAG 跑完后，由查询接口返回真实数据，而不是让 LLM 自己编：

java

// DeviceTools.queryDeviceStatus — AI 查进度时调用@Tool(description = "查询设备申领进度")public String queryDeviceStatus(@ToolParam String requestNo) {

AdminDeviceRequest request = requestMapper.selectOne(

new QueryWrapper\<AdminDeviceRequest\>().eq("request_no", requestNo)

);

if (request == null) return "未找到该申领单";

// 只有 DAG 跑完后，allocated_device_id 才有值

if (request.getStatus() == 4 && request.getAllocatedDeviceId() != null) {

return String.format(

"申领单 %s 状态：待领取，已分配设备 ID: %d",

requestNo, request.getAllocatedDeviceId() // ← 真实数据库数据

);

}

// ...}

核心原则：LLM 只负责"转述"工具返回的数据，不负责"创造"数据。

第三步（可选）：输出层加校验

在 LLM 响应返回给用户之前，用正则提取 "ID:(\\d+)"，然后校验这个数字是否在工具返回的数据中出现过。如果没出现，就替换为 "未获取到" 或直接拦截重试。

修改过提示词后：

<img src="media/image2.png" style="width:5.76458in;height:1.42222in" />

五、总结：AI 应用开发的"不信任原则"

这个 Bug 让我对 LLM 应用开发有了更深的认知：

表格

| 认知                   | 说明                                                          |
|------------------------|---------------------------------------------------------------|
| LLM 不是数据库         | 它不会查表，它只是"猜测"下一个最合理的词                      |
| LLM 会"自信地胡说"     | 编造的数字看起来完全合理（15、16、17 连续递增），但完全是假的 |
| Prompt 约束要具体      | "不要编造"太笼统，必须明确列出哪些字段不能编                  |
| 关键数据必须由后端提供 | 设备 ID、金额、状态等客观事实，必须从数据库查询后注入         |

在 AI 应用开发中，我总结了一条原则："不信任 LLM 的输出，只信任工具返回的数据。"

LLM 适合做的是：理解用户意图、调用正确工具、格式化展示结果。但它不适合做的是：生成客观事实、做数学计算、提供精确编号。

如果你也在做 LLM + 工具调用（Function Calling）的项目，建议检查一下：你的 System Prompt 里有没有明确禁止 LLM 编造时间、ID、金额、数量这类客观数据？没有的话，它可能正在"自信地胡说"。
