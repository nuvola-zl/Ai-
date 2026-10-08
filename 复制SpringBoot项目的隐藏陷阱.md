# 复制 Spring Boot 项目时，那些看不见的”身份残留”

> 刚接触 Spring Boot 那会儿，我以为”复制文件夹 = 复制项目”。直到踩了一次坑才明白：工具链识别项目，看的从来不是外层文件夹的名字。

## 背景

学 Spring Boot 三个月后，我写了第一个练手项目——**TradeHub**，一个校园二手交易平台。技术栈很常规：Spring Boot + MyBatis + MySQL。

不久后，我想再写一个功能类似的项目——**SwapHub**，校园闲置交换平台。两个项目都是 C2C 交易逻辑，框架几乎一模一样。为了”省事”，我直接 `cp -r` 复制了 TradeHub 的目录，重命名为 SwapHub，然后把 `src/` 里的业务代码删了个干净，只保留了框架和配置。

我自信满满地打开 IDEA，以为这是一个”全新”的项目。然而很快发现了不对劲：

-   IDEA 的窗口标题偶尔还会闪过 TradeHub 的字样；
-   打包出来的 JAR 文件名还是 `tradehub-0.0.1-SNAPSHOT.jar`；
-   更诡异的是，数据库连接配置里居然还残留着 `tradehub` 的 schema 名称。

我明明已经删光了业务代码，为什么工具还是”认”出了 TradeHub？

## 排查：不是代码的问题，是”身份”的问题

首先检查的是明面上的配置：

-   `pom.xml` 里的 `<artifactId>` 和 `<name>` 还是 `tradehub`；
-   `application.yml` 里的 `spring.application.name` 还是 `TradeHub`。

这些改起来很快，但改完之后，某些工具依然表现得像是面对 TradeHub。我意识到问题不在业务代码，而在那些**平时看不见的”项目元数据”**。

## 深层原因：三个”隐形身份证”

### 1. `.git`——项目的”血缘档案”

很多人（包括当时的我）对 `.git` 的理解停留在”它帮我存代码历史”。但实际上，`.git` 是一个**完整的本地数据库**，里面不仅存了每一次提交的哈希、作者、时间戳，还存了**远程仓库的地址**、分支映射、标签、钩子脚本等。

当你用 `cp -r` 或系统自带的复制粘贴时，操作系统是**递归复制**的，`.git` 这个隐藏文件夹会被原封不动地带过去。这意味着 SwapHub 目录下的 `.git/config` 里，`remote.origin.url` 依然指向 TradeHub 的 GitHub 地址。

换句话说：**SwapHub 在 Git 的眼里，本质上还是 TradeHub 的一个本地副本。**

### 2. `.idea`——IDE 的”记忆库”

`.idea` 是 IntelliJ IDEA 的本地项目配置目录。它不存你的业务代码，但存了 IDEA 对这个项目的**全部记忆**：

-   `modules.xml` 和 `.iml` 文件里的模块名；
-   `dataSources.xml` 里记录的数据库连接（包括 URL、用户名、甚至 schema 名）；
-   `vcs.xml` 里的版本控制映射路径；
-   `workspace.xml` 里你上次打开的文件、断点位置、运行配置等。

复制项目时，`.idea` 跟着一起过来，IDEA 打开 SwapHub 时，读取的还是 TradeHub 的”记忆”。这就解释了为什么数据源配置里还残留着旧项目的 schema。

### 3. 构建文件里的”隐式项目名”

除了隐藏文件夹，构建文件里也有一些容易被忽略的”身份标识”：

-   Maven 的 `pom.xml`：`<artifactId>` 决定 artifact 名，`<name>` 决定 IDE 显示名；
-   Gradle 的 `settings.gradle`：`rootProject.name` 决定项目根名；
-   Spring Boot 的 `application.yml`：`spring.application.name` 会注册到服务发现、监控、日志等组件中。

这些名字不会因为你改了外层文件夹名而自动变化。

## 为什么改文件夹名没用？

操作系统层面的文件夹名，只是文件系统的一个**容器标签**。但 Maven、Gradle、Git、IDEA 这些工具链在识别项目时，读的是**内部元数据**。

打个比方：你把身份证上的名字用贴纸盖住了，但公安系统里的户籍记录没变。工具链就是那个”公安系统”，它们认的是 `.git/config`、`pom.xml`、`.idea/.name` 里的记录，而不是你文件夹叫什么。

## 正确的做法：复制骨架，不复制身份

如果你也想基于旧项目的框架快速启动新项目，建议按以下清单执行清理：

`# 1. 删除旧的 Git 仓库（最关键）`  
`rm`` ``-rf`` .git`  
  
`# 2. 删除 IDEA 本地配置`  
`rm`` ``-rf`` .idea`  
`rm`` ``-f`` ``*``.iml`  
  
`# 3. 删除编译产物`  
`rm`` ``-rf`` target/`  
`rm`` ``-rf`` build/`  
  
`# 4. 重新初始化 Git`  
`git`` init`  
`git`` add .`  
`git`` commit ``-m`` ``"init: 新项目初始化"`

然后手动修改以下文件中的项目标识：

-   `pom.xml` → `<artifactId>`、`<name>`
-   `settings.gradle`（如适用）→ `rootProject.name`
-   `application.yml` / `application.properties` → `spring.application.name`

最后，**用 IDEA 重新打开项目文件夹**，让它生成一套全新的 `.idea` 配置。

## 更进一步：做成项目模板

这次踩坑让我意识到，“复制项目”和”复用框架”是两回事。如果团队或个人经常需要基于同一套技术栈开新项目，更好的做法是维护一个**干净的框架模板**（Template）。

把清理后的空壳项目推到一个独立的 Git 仓库，比如 `springboot-base-template`。以后新项目直接：

`git`` clone https://github.com/yourname/springboot-base-template.git 新项目名`  
`cd`` 新项目名`  
`rm`` ``-rf`` .git`  
`git`` init`  
`# 修改 pom.xml 和 application.yml 中的项目名，即可开始开发`

这样既复用了依赖配置和目录结构，又彻底避免了旧项目的”身份残留”。

## 总结

刚入门时，我们往往只关注 `src/` 里的代码，却忽略了项目背后那张由 `.git`、`.idea`、构建文件共同构成的”身份网络”。理解工具链如何识别一个项目，是工程化思维的重要一步。

现在每次启动新项目，我都会先执行上面的初始化清单。毕竟，**干净的起点，比省下的那几分钟复制时间重要得多。**
