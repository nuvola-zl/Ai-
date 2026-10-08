# 从 RabbitMQ 报错到理解 Spring AMQP 3.x 反序列化安全机制

> **项目背景**：ShareShelf 教材共享平台（课设/毕设项目），基于 Spring Boot 3.x + RabbitMQ 实现异步申领入库流程。 **记录时间**：2026-08-23 **关键词**：Spring AMQP、RabbitMQ、Jackson2JsonMessageConverter、反序列化白名单

## 写在前面

这篇笔记不是一篇干巴巴的教程，而是我在做课设时**真实踩坑的记录**。我想把”发现问题 → 一脸懵 → 查资料 → 恍然大悟 → 解决”的完整心路历程写出来。如果你也在用 Spring Boot + RabbitMQ，希望这篇笔记能帮你少走点弯路。

## 第一章：发现问题 —— 启动就报错

事情是这样的。我的 borrow（借阅）模块已经写好了异步申领逻辑：用户申请热门教材时，先 Redis 预扣库存，再发 MQ 异步落库。代码看起来挺完美的，结果一启动……

`java.lang.SecurityException: Attempt to deserialize unauthorized class `  
`com.shelf.borrow.domain.dto.BorrowApplyMessage; `  
`add allowed class name patterns to the message converter or, `  
`if you trust the message originator, set environment variable `  
`'SPRING_AMQP_DESERIALIZATION_TRUST_ALL' or system property `  
`'spring.amqp.deserialization.trust.all' to true`

**我当时的内心 OS**：

> “啥？我自己的类，我自己写的项目，你告诉我 unauthorized？RabbitMQ 你礼貌吗？”

更离谱的是，这个报错不是发消息时报的，是**消费者一启动就报**。也就是说，应用刚跑起来，Listener 还没开始正经干活，就躺平了。

## 第二章：探索问题 —— 先别慌，看堆栈

我冷静下来（其实没冷静），把报错堆栈从头到尾看了一遍。

### 2.1 定位关键类

堆栈里反复出现一个类：

`org.springframework.amqp.utils.SerializationUtils.checkAllowedList`

顺着点进去，发现 Spring AMQP 在反序列化消息前，会先检查这个类是不是在”允许列表”里。如果不在，直接抛 `SecurityException`。

### 2.2 我的第一个”修复”（错的）

报错信息很”贴心”地给了个捷径：

> set environment variable `SPRING_AMQP_DESERIALIZATION_TRUST_ALL` or system property `spring.amqp.deserialization.trust.all` to true

我心想：“这不简单？” 于是直接在 `application.yml` 里加了：

`spring``:`  
`  ``amqp``:`  
`    ``deserialization``:`  
`      ``trust``:`  
`        ``all``:`` ``true`

**结果：没用，照样报错。**

### 2.3 为什么 YAML 配置不生效？

我又去翻了源码，发现 `SerializationUtils` 里是这样读的：

`static`` ``{`  
`    ``String`` trustedPackages ``=`` ``System``.``getProperty``(``"spring.amqp.deserialization.trust.all"``);`  
`    ``// ...`  
`}`

**注意**：它读的是 **System Property**（JVM 启动参数），不是 Spring Boot 的 Environment。YAML 里的配置此时可能还没加载完，或者根本就不是一个配置路径。

也就是说，这个开关设计出来就不是让你写在配置文件里的，而是**应急用的系统属性**。

**我当时的内心 OS**：

> “好家伙，报错信息误导性有点强啊……”

## 第三章：理解问题 —— 这不是 Bug，是安全机制

既然配置绕不过去，我开始认真思考：**为什么 Spring 要拦我？**

### 3.1 Java 原生序列化的安全隐患

我之前一直用 `Serializable` + `SimpleMessageConverter`，也就是 Java 原生的 `ObjectInputStream` 反序列化。这种方式有个致命问题：**如果消息被篡改，可以注入恶意对象，触发远程代码执行（RCE）**。

Spring AMQP 3.x 显然意识到了这一点，所以加了 `AllowedListDeserializingMessageConverter`，默认只允许反序列化 `java.util.*`、`java.lang.*` 等可信包下的类。

### 3.2 核心矛盾

我的 `BorrowApplyMessage` 在 `com.shelf.borrow.domain.dto` 包下，自然不在白名单里。于是冲突产生了：

-   **框架**：为了安全，我不认识你，我不让你反序列化。
-   **我**：这是我自己写的类啊！

**但框架没法判断”你是不是真的信任这个类”**，所以它选择默认拒绝。

### 3.3 顿悟时刻

> “绕过安全检查不是解决问题，**换掉有问题的序列化方式才是**。

Java 原生序列化本来就不适合微服务场景。我的消息是要在 RabbitMQ 里流转的，万一以后有其他语言的服务要消费呢？二进制格式别人根本读不了。

**JSON 才是跨语言、跨服务的标准契约。**

## 第四章：解决问题 —— 统一 JSON 序列化

### 4.1 方案对比

| 方案 | 做法                                                           | 评价                                            |
|------|----------------------------------------------------------------|-------------------------------------------------|
| A    | 加 JVM 启动参数 `-Dspring.amqp.deserialization.trust.all=true` | 能跑，但等于关闭安全防护，**生产环境大忌**      |
| B    | 配置 `SimpleMessageConverter` 的 `allowedListPatterns`         | 稍微好点，但还在用 Java 原生序列化，治标不治本  |
| C    | **换成** `Jackson2JsonMessageConverter`                        | **根治**，JSON 格式通用、可读、无安全白名单问题 |

我选了方案 C。

### 4.2 具体实施

#### 步骤 1：common 模块新增配置

因为 borrow 和 donate 模块都要发/收 MQ，我把转换器下沉到 common 模块，避免重复配置。

`package`` com``.``shelf``.``common``.``config``;`  
  
`import`` ``org``.``springframework``.``amqp``.``support``.``converter``.``Jackson2JsonMessageConverter``;`  
`import`` ``org``.``springframework``.``amqp``.``support``.``converter``.``MessageConverter``;`  
`import`` ``org``.``springframework``.``context``.``annotation``.``Bean``;`  
`import`` ``org``.``springframework``.``context``.``annotation``.``Configuration``;`  
  
`@Configuration`  
`public`` ``class`` MqConfig ``{`  
  
`    ``@Bean`  
`    ``public`` MessageConverter ``jsonMessageConverter``()`` ``{`  
`        ``return`` ``new`` ``Jackson2JsonMessageConverter``();`  
`    ``}`  
`}`

然后在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 里加上：

`com.shelf.common.config.MqConfig`

这样所有引入 common 的模块启动时自动加载，**零侵入**。

#### 步骤 2：DTO 去掉 Serializable

既然不用 Java 原生序列化了，`Serializable` 和 `serialVersionUID` 就是纯粹的”历史包袱”。

**改造前：**

`@Data`  
`public`` ``class`` BorrowApplyMessage ``implements`` ``Serializable`` ``{`  
`    ``@Serial`  
`    ``private`` ``static`` ``final`` ``long`` serialVersionUID ``=`` ``1L``;`  
`    ``private`` ``String`` recordNo``;`  
`    ``private`` ``Long`` userId``;`  
`    ``private`` ``String`` isbn``;`  
`    ``private`` ``String`` requestId``;`  
`}`

**改造后：**

`@Data`  
`public`` ``class`` BorrowApplyMessage ``{`  
`    ``private`` ``String`` recordNo``;`  
`    ``private`` ``Long`` userId``;`  
`    ``private`` ``String`` isbn``;`  
`    ``private`` ``String`` requestId``;`  
`}`

清爽多了。

#### 步骤 3：清理 YAML 里的无效配置

把之前那个不生效的 `trust.all` 从 `application.yml` 里删掉，眼不见为净。

## 第五章：踩坑 —— 旧消息怎么办？

我信心满满地重启服务，结果又报错：

`Could not convert incoming message with content-type `  
`[application/x-java-serialized-object], 'json' keyword missing.`

**我当时的内心 OS**：

> “不是已经换 Jackson 了吗？怎么还报？”

### 5.1 原因

RabbitMQ 队列里**还躺着之前用 Java 原生序列化发的旧消息**。它们的 `contentType` 是 `application/x-java-serialized-object`，而 `Jackson2JsonMessageConverter` 只认带 `json` 关键字的内容类型。

消费者一启动就拉到了这条”古董消息”，JSON 转换器一看：“这啥？我不认识。” 直接抛异常。

### 5.2 解决

登录 RabbitMQ 管理界面（`http://ip:15672`），找到相关队列：

-   `borrow.apply.queue`
-   `borrow.apply.dlx.queue`
-   `donate.inbound.queue`
-   `donate.inbound.dlx.queue`

**先停掉应用**（让 Unacked 的消息回到 Ready），然后点击 **Purge** 清空，或者直接 **Delete** 队列（Spring Boot 会自动重建）。

重启应用，世界安静了。

## 第六章：复盘 —— 这件事教会了我什么

### 6.1 技术层面

1.  **Spring AMQP 3.x 的安全白名单机制**：不是 Bug，是 feature。Java 原生序列化在分布式场景下确实该被淘汰。
2.  **消息契约应该语言无关**：JSON 是微服务间通信的更好选择。
3.  **系统属性 ≠ 配置属性**：`spring.amqp.deserialization.trust.all` 是 JVM 系统属性，不是 Spring Boot 配置项，写在 YAML 里大概率不生效。

### 6.2 工程层面

1.  **common 模块下沉基础设施**：消息转换器这种全局能力，应该统一维护，而不是每个服务各写一份。
2.  **切换序列化格式时要清理队列**：这是一个很容易忽略的迁移成本。消息一旦进队列，格式就定死了。
3.  **不要看到** `trust.all` **就用**：框架给的安全后门，往往是让你应急排查的，不是让你长期开着的。

### 6.3 面试价值

如果面试官问：“你在项目里遇到过什么棘手的问题？”

我可以这样回答：

> “我在做异步申领模块时，遇到了 Spring AMQP 3.x 的反序列化白名单限制。我的第一反应是配 `trust.all` 绕过，但发现 YAML 不生效，深入源码后意识到它读的是 JVM 系统属性。
>
> 更重要的是，我理解到 Java 原生序列化在微服务场景下的安全隐患，所以选择了根治方案：统一换成 `Jackson2JsonMessageConverter`，下沉到 common 模块自动装配。迁移过程中还踩了旧消息格式不兼容的坑，通过清理队列解决。
>
> 这件事让我对框架的安全设计有了更深的理解，也意识到消息契约应该选择跨语言的格式。”

## 结语

做课设/毕设的意义，不只是把功能做出来，而是在踩坑的过程中真正理解技术背后的设计思想。这个报错表面上是一个配置问题，本质上是**分布式系统中消息契约设计**的问题。

希望这篇笔记对你有帮助。如果你也遇到了类似的报错，别急着配 `trust.all`，停下来想想：**框架为什么拦你？有没有更好的做法？**

有时候，绕路才是最快的路。

*记录于 ShareShelf 项目开发期间。*
