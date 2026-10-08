# 把项目推上 GitHub 之前：一份来自真实踩坑的敏感信息治理指南

> 项目：Haze AI Hub（Spring Boot 3.2 + Spring AI + PostgreSQL + Redis）  
> 背景：准备将公司内部 AI 助手平台脱敏后上传至 GitHub，作为面试作品集展示。

## 一、灾难预演：当我打开 `application.yml` 时，手心冒汗了

在准备把项目推上 GitHub 的前一晚，我按照 checklist 逐项检查配置文件。当我打开 `application.yml` 时，看到了这样一段代码——**这要是直接传上去，面试还没开始，安全漏洞先出名了**。

`# ❌ 反面教材：绝对不能直接上传 GitHub 的 application.yml`  
`spring``:`  
`  ``application``:`  
`    ``name``:`` haze-ai-hub`  
`  ``profiles``:`  
`    ``active``:`` dev`  
`  ``ai``:`  
`    ``dashscope``:`  
`      ``api-key``:`` sk-d95a5e03abcdefghigklmn``  # 阿里云灵积 API Key`  
`      ``chat``:`  
`        ``options``:`  
`          ``model``:`` deepseek-v3`  
`          ``temperature``:`` ``0.8`  
`      ``embedding``:`  
`        ``options``:`  
`          ``model``:`` text-embedding-v3`  
`    ``wanx``:`  
`      ``model``:`` wanxFLUX`  
`      ``endpoint``:`` https://dashscope.aliyuncs.com/api/v1/services/a2xlvm9xb7tx/image%20generation`  
`      ``timeout``:`` ``60000`  
  
`  ``datasource``:`  
`    ``url``:`` jdbc:postgresql://192.168.153.129:5432/assistant?sslmode=disable``  # 内网数据库IP`  
`    ``driver-class-name``:`` org.postgresql.Driver`  
`    ``username``:`` postgres`  
`    ``password``:`` 032807``  # 数据库明文密码`  
  
`  ``data``:`  
`    ``redis``:`  
`      ``host``:`` ``192.168.153.129``  # 内网Redis地址`  
`      ``port``:`` ``6379`  
`      ``password``:``               # 即使为空，也暴露了架构信息`  
`      ``timeout``:`` ``10000`  
`      ``database``:`` ``0`

**这段配置泄露了什么？**

| 泄露项                           | 风险等级    | 后果                                           |
|----------------------------------|-------------|------------------------------------------------|
| `api-key: sk-d95a5e03...`        | 🔴 **致命** | 阿里云 AK 泄露，攻击者可直接调用模型，账单暴涨 |
| `192.168.153.129`                | 🟠 **高危** | 内网 IP 暴露，配合其他漏洞可直接渗透           |
| `password: 032807`               | 🟠 **高危** | 数据库密码明文，脱库风险                       |
| `endpoint: .../a2xlvm9xb7tx/...` | 🟡 **中危** | 暴露了内部服务路径和项目 ID                    |
| `database: 0` + `timeout: 10000` | 🟢 **低危** | 架构信息泄露，辅助攻击者画像                   |

> 注：文中 `api-key` 为演示用的假值，但**即使是假的，也不应该出现在仓库里**——因为 Git 历史会永久记录你的修改痕迹，攻击者可以通过 `git log` 看到旧版本里的真实密钥。

## 二、为什么 “我后面会删” 没用？Git 的永恒记忆

很多新手有个误区：“我先传上去，后面把密码改成 `******` 再提交一次不就行了？”

**不行。Git 会记住每一次修改。**

`# 攻击者只需要执行这一行，就能看到你第一次提交时的真实密码`  
`git`` log ``-p`` ``--`` application.yml`  
  
`# 或者更直接地查看某个文件的历史版本`  
`git`` show HEAD~3:src/main/resources/application.yml`

只要密钥曾经出现在某个 commit 里，它就**永远留在了 Git 历史中**。即使你删除了文件、修改了内容，那些敏感信息依然可以通过 `git log` 找回。

**真实案例**：2023 年某大厂实习生将内部 AK 上传至 GitHub，虽然 10 分钟后就删除了文件，但已经被 GitHub 的 Secret Scanning 抓取并告警，云厂商在 5 分钟内冻结了账号，导致生产环境服务中断 2 小时。

## 三、方案一：环境变量 + `${}` 占位符（最标准）

Spring Boot 原生支持 `${}` 语法读取环境变量。这是**企业级项目最标准的做法**。

### 3.1 改造后的 `application.yml`（可安全上传）

`# ✅ 正面教材：使用占位符，无敏感信息，可上传 GitHub`  
`spring``:`  
`  ``application``:`  
`    ``name``:`` haze-ai-hub`  
`  ``profiles``:`  
`    ``active``:`` ${SPRING_PROFILES_ACTIVE:dev}``  # 默认 dev，可通过环境变量覆盖`  
  
`  ``ai``:`  
`    ``dashscope``:`  
`      ``api-key``:`` ${DASHSCOPE_API_KEY:}``        # 从环境变量读取，无默认值`  
`      ``chat``:`  
`        ``options``:`  
`          ``model``:`` deepseek-v3`  
`          ``temperature``:`` ``0.8`  
`      ``embedding``:`  
`        ``options``:`  
`          ``model``:`` text-embedding-v3`  
`    ``wanx``:`  
`      ``model``:`` wanxFLUX`  
`      ``endpoint``:`` ${WANX_ENDPOINT:https://dashscope.aliyuncs.com/api/v1/services/aigc/text2image}`  
`      ``timeout``:`` ``60000`  
  
`  ``datasource``:`  
`    ``url``:`` ${DB_URL:jdbc:postgresql://localhost:5432/assistant?sslmode=disable}`  
`    ``driver-class-name``:`` org.postgresql.Driver`  
`    ``username``:`` ${DB_USERNAME:postgres}`  
`    ``password``:`` ${DB_PASSWORD:}``               # 空默认值，强制外部传入`  
  
`  ``data``:`  
`    ``redis``:`  
`      ``host``:`` ${REDIS_HOST:localhost}`  
`      ``port``:`` ${REDIS_PORT:6379}`  
`      ``password``:`` ${REDIS_PASSWORD:}`  
`      ``timeout``:`` ``10000`  
`      ``database``:`` ``0`

**关键语法**： - `${VAR_NAME}` —— 读取环境变量，不存在则启动报错 - `${VAR_NAME:default}` —— 读取环境变量，不存在则使用默认值 - `${VAR_NAME:}` —— 读取环境变量，不存在则为空字符串（适合密码字段）

### 3.2 在 IDEA 中配置环境变量（本地开发）

不需要每次启动都在终端 `export`，IDEA 提供了图形化配置：

**步骤**： 1. 点击右上角运行配置下拉框 → `Edit Configurations...` 2. 选择你的 Spring Boot 启动项（如 `HazeAiHubApplication`） 3. 右侧找到 `Environment variables:` 输入框 4. 点击右侧文件夹图标，添加键值对：

`DASHSCOPE_API_KEY=sk-d95a5e03abcdefghigklmn`  
`DB_URL=jdbc:postgresql://192.168.153.129:5432/assistant?sslmode=disable`  
`DB_USERNAME=postgres`  
`DB_PASSWORD=032807`  
`REDIS_HOST=192.168.153.129`  
`REDIS_PASSWORD=`

5.  点击 `Apply` → `OK`

**效果**： - 代码仓库里没有任何敏感信息 - 每个开发者用自己的本地环境变量，互不干扰 - 生产服务器通过启动脚本注入环境变量，运维独立管理

### 3.3 启动脚本示例（生产环境）

`#!/bin/bash`  
`# start.sh —— 放在服务器上，不在 Git 中`  
  
`export`` ``DASHSCOPE_API_KEY``=``"sk-xxxxxxxxxxxx"`  
`export`` ``DB_URL``=``"jdbc:postgresql://prod-db.internal:5432/assistant?sslmode=require"`  
`export`` ``DB_PASSWORD``=``"``$(``cat`` /etc/secrets/db_password``)``"``  ``# 从文件读取，更安全`  
`export`` ``REDIS_HOST``=``"prod-redis.internal"`  
  
`java`` ``-jar`` haze-ai-hub.jar   ``--spring.profiles.active``=``prod`

## 四、方案二：`.env` 文件 + `.gitignore`（本地开发最优雅）

环境变量在 IDEA 里配置有个缺点：**换电脑要重新配**。更优雅的做法是使用 `.env` 文件。

### 4.1 项目结构

`haze-ai-hub/`  
`├── .gitignore                    # 忽略 .env`  
`├── .env.example                  # ✅ 示例文件，可上传，供队友参考`  
`├── .env                          # ❌ 真实配置，不上传`  
`├── pom.xml`  
`└── src/`  
`    └── main/`  
`        └── resources/`  
`            └── application.yml   # 读取 ${} 占位符`

### 4.2 `.env` 文件（真实值，不上传）

`# .env —— 这个文件在 .gitignore 里，永远不进仓库`  
`DASHSCOPE_API_KEY``=``sk-d95a5e03abcdefghigklmn`  
`DB_URL``=``jdbc:postgresql://192.168.153.129:5432/assistant``?``sslmode=disable`  
`DB_USERNAME``=``postgres`  
`DB_PASSWORD``=``032807`  
`REDIS_HOST``=``192.168.153.129`  
`REDIS_PASSWORD``=`

### 4.3 `.env.example` 文件（模板，可上传）

`# .env.example —— 复制一份为 .env 后填入真实值`  
`DASHSCOPE_API_KEY``=``your-dashscope-api-key`  
`DB_URL``=``jdbc:postgresql://localhost:5432/assistant``?``sslmode=disable`  
`DB_USERNAME``=``postgres`  
`DB_PASSWORD``=``your-password`  
`REDIS_HOST``=``localhost`  
`REDIS_PASSWORD``=`

### 4.4 `.gitignore` 核心规则

`# ========== 敏感配置（红线） ==========`  
`.env`  
`.env.local`  
`.env.production`  
  
`# 生产环境配置文件`  
`application-prod.yml`  
`application-production.yml`  
  
`# 密钥文件`  
`*.pem`  
`*.key`  
`*.p12`  
`*.jks`  
  
`# 任何包含 secret/private 字样的文件`  
`*secret*`  
`*private*`

### 4.5 如何让 Spring Boot 读取 `.env`？

Spring Boot 本身不直接支持 `.env` 文件，但可以通过以下方式：

**方式 A：IDEA 插件（推荐开发环境）** - 安装插件 `.env files support` - IDEA 会自动识别项目根目录的 `.env` 文件，并将其中的变量注入到运行环境

**方式 B：命令行加载（通用）**

`# Linux/Mac`  
`export`` ``$(``cat`` .env ``|`` ``xargs``)`` ``&&`` ``java`` ``-jar`` haze-ai-hub.jar`  
  
`# Windows PowerShell`  
`Get-Content`` .env ``|`` ``ForEach-Object`` { if ``(``$_`` ``-match`` ``"^([^#][^=]*)=(.*)$"``)`` ``{`` ``[Environment]::SetEnvironmentVariable``(``$matches``[1],`` ``$matches``[``2``]``)`` ``}`` ``}``;`` ``java`` ``-jar`` haze-ai-hub.jar`

**方式 C：引入 dotenv-java 库（代码层面）**

`<``dependency``>`  
`    <``groupId``>io.github.cdimascio</``groupId``>`  
`    <``artifactId``>dotenv-java</``artifactId``>`  
`    <``version``>3.0.0</``version``>`  
`</``dependency``>`

`// 在启动类中加载`  
`import`` ``io``.``github``.``cdimascio``.``dotenv``.``Dotenv``;`  
  
`@SpringBootApplication`  
`public`` ``class`` HazeAiHubApplication ``{`  
`    ``public`` ``static`` ``void`` ``main``(``String``[]`` args``)`` ``{`  
`        ``// 仅在 dev 环境加载 .env`  
`        ``if`` ``(``System``.``getenv``(``"SPRING_PROFILES_ACTIVE"``)`` ``==`` ``null``)`` ``{`  
`            Dotenv dotenv ``=`` Dotenv``.``configure``().``ignoreIfMissing``().``load``();`  
`            dotenv``.``entries``().``forEach``(``e ``->`` ``System``.``setProperty``(``e``.``getKey``(),`` e``.``getValue``()));`  
`        ``}`  
`        SpringApplication``.``run``(``HazeAiHubApplication``.``class``,`` args``);`  
`    ``}`  
`}`

> 注意：生产环境不要加载 `.env`，应该通过容器编排（K8s Secret、Docker Secret）或环境变量注入。

## 五、方案三：多环境配置分离（企业级标准）

对于面试作品集，最专业的方式是展示**多环境配置分离**。

### 5.1 文件结构

`src/main/resources/`  
`├── application.yml              # 通用配置，无敏感信息，可上传`  
`├── application-dev.yml          # 开发环境，使用本地 Docker，可上传`  
`├── application-test.yml         # 测试环境，可上传`  
`└── application-prod.yml         # ❌ 生产环境，包含真实密钥，不上传`

### 5.2 `application.yml`（通用模板）

`spring``:`  
`  ``profiles``:`  
`    ``active``:`` ${SPRING_PROFILES_ACTIVE:dev}`  
  
`  ``ai``:`  
`    ``dashscope``:`  
`      ``api-key``:`` ${DASHSCOPE_API_KEY:}``  # 强制外部传入`  
`      ``chat``:`  
`        ``options``:`  
`          ``model``:`` deepseek-v3`

### 5.3 `application-dev.yml`（开发环境，可上传）

`# 开发环境使用本地 Docker，无敏感信息`  
`spring``:`  
`  ``datasource``:`  
`    ``url``:`` jdbc:postgresql://localhost:5432/assistant?sslmode=disable`  
`    ``username``:`` postgres`  
`    ``password``:`` postgres`  
  
`  ``data``:`  
`    ``redis``:`  
`      ``host``:`` localhost`  
`      ``port``:`` ``6379`  
`      ``password``:`

### 5.4 `application-prod.yml`（生产环境，不上传）

`# 这个文件只在服务器上存在，通过 --spring.config.location 加载`  
`spring``:`  
`  ``datasource``:`  
`    ``url``:`` jdbc:postgresql://192.168.153.129:5432/assistant?sslmode=disable`  
`    ``username``:`` postgres`  
`    ``password``:`` 032807`  
  
`  ``data``:`  
`    ``redis``:`  
`      ``host``:`` ``192.168.153.129`

### 5.5 启动方式

`# 开发环境（本地）`  
`java`` ``-jar`` haze-ai-hub.jar`  
`# 或 IDEA 直接运行，自动读取 application-dev.yml`  
  
`# 生产环境（服务器）`  
`java`` ``-jar`` haze-ai-hub.jar   ``--spring.profiles.active``=``prod   ``--spring.config.location``=``/opt/haze-ai-hub/config/`

## 六、完整 `.gitignore`（Java Spring Boot 项目标准版）

`# ========== 编译产物 ==========`  
`target/`  
`build/`  
`*.class`  
`*.jar`  
`*.war`  
`*.ear`  
  
`# ========== IDE ==========`  
`.idea/`  
`*.iml`  
`*.ipr`  
`*.iws`  
`.classpath`  
`.project`  
`.settings/`  
`.vscode/`  
`*.suo`  
`*.ntvs*`  
`*.njsproj`  
`*.sln`  
`*.sw?`  
  
`# ========== 系统文件 ==========`  
`.DS_Store`  
`Thumbs.db`  
`*.tmp`  
`*.bak`  
`*.log`  
  
`# ========== 敏感配置（核心红线） ==========`  
`# 环境变量文件`  
`.env`  
`.env.local`  
`.env.production`  
`.env.*`  
  
`# 生产环境配置文件`  
`application-prod.yml`  
`application-production.yml`  
`application-prd.yml`  
  
`# 密钥文件`  
`*.pem`  
`*.key`  
`*.p12`  
`*.jks`  
`*.keystore`  
`*.truststore`  
  
`# 任何包含敏感字样的文件`  
`*secret*`  
`*private*`  
`*password*`  
`*credential*`  
  
`# ========== 本地数据 ==========`  
`logs/`  
`uploads/`  
`*.db`  
`*.sqlite`  
`h2-db/`  
  
`# ========== Maven/Gradle ==========`  
`.mvn/`  
`mvnw`  
`mvnw.cmd`  
`.gradle/`  
`gradle/`  
`gradlew`  
`gradlew.bat`  
`!gradle/wrapper/gradle-wrapper.jar`  
  
`# ========== 其他 ==========`  
`node_modules/`

## 七、上传前的检查清单（Checklist）

在点击 `git push` 之前，逐项确认：

| 检查项                                         | 命令/方法                                 | 是否通过 |
|------------------------------------------------|-------------------------------------------|----------|
| 1\. 确认没有硬编码的 API Key                   | `grep -r "sk-" src/`                      | ☐        |
| 2\. 确认没有明文密码                           | `grep -r "password:" src/main/resources/` | ☐        |
| 3\. 确认没有内网 IP                            | `grep -r "192\.168\." src/`               | ☐        |
| 4\. `.env` 已加入 `.gitignore`                 | `cat .gitignore \| grep "\.env"`          | ☐        |
| 5\. `application-prod.yml` 已加入 `.gitignore` | `cat .gitignore \| grep "prod"`           | ☐        |
| 6\. 提交前查看变更文件                         | `git status`                              | ☐        |
| 7\. 确认 diff 中没有敏感信息                   | `git diff --cached`                       | ☐        |
| 8\. README 中说明了如何配置环境变量            | 查看 README.md                            | ☐        |

## 八、如果不小心上传了，如何补救？

### 8.1 立刻做的三件事（黄金 5 分钟）

1.  **撤销密钥**：

    -   阿里云控制台 → AccessKey 管理 → 禁用并删除泄露的 AK
    -   数据库管理后台 → 修改密码
    -   JWT Secret → 重新生成并替换

2.  **清理 Git 历史**（必须做，否则等于没删）：

-   `# 安装 git-filter-repo（比 git filter-branch 更快更安全）`  
    `pip`` install git-filter-repo`  
      
    `# 彻底删除 application-prod.yml 的所有历史记录`  
    `git`` filter-repo ``--path`` src/main/resources/application-prod.yml ``--invert-paths`  
      
    `# 强制推送到远程（会重写历史，团队项目需提前通知队友）`  
    `git`` push origin ``--force`` ``--all`

3.  **检查 GitHub Security 告警**：

    -   进入仓库 → `Settings` → `Security` → `Secret scanning alerts`
    -   查看是否有被 GitHub 自动扫描到的密钥

### 8.2 千万不要做的（常见误区）

| 错误做法                              | 为什么没用                              |
|---------------------------------------|-----------------------------------------|
| `git rm application.yml` 然后重新提交 | 文件仍在历史记录中                      |
| 修改密码后重新提交                    | 旧密码仍在 `git log` 里                 |
| 把仓库设为 Private                    | 历史记录仍在，且 Private 不代表绝对安全 |
| 删除仓库重新创建                      | 如果已被 fork 或缓存，依然泄露          |

## 九、面试时的加分回答

如果面试官问”你的项目怎么管理敏感配置”：

> “我在 Haze AI Hub 项目里，配置文件做了三层安全隔离：
>
> **第一层是代码层**：`application.yml` 里所有敏感字段都用 `${}` 占位符，比如 `api-key: ${DASHSCOPE_API_KEY:}`、`password: ${DB_PASSWORD:}`，代码仓库里没有任何真实密钥。
>
> **第二层是环境层**：本地开发使用 `.env` 文件管理环境变量，并通过 `.gitignore` 确保它不进 Git；生产环境通过服务器启动脚本注入环境变量，运维独立管理。
>
> **第三层是历史层**：我配置了 `git filter-repo` 的预提交检查，确保即使误操作也能快速清理历史。同时仓库启用了 GitHub Secret Scanning，一旦检测到类似 `sk-` 开头的阿里云 AK，会自动告警。
>
> 这样保证代码仓库是’干净’的，可以安全地作为面试作品集展示，同时生产环境的密钥与代码完全解耦。”

## 十、总结：安全上传的三条铁律

1.  **代码里永远没有真实密钥**：用 `${}` 占位符，用 `.env` 管理本地环境，用服务器脚本管理生产环境。
2.  `.gitignore` **是底线，不是保险**：即使加了 `.gitignore`，提交前也要执行 `git status` 和 `git diff` 二次确认。
3.  **历史记录比当前文件更危险**：Git 会记住一切，泄露后必须清理历史，否则等于没删。

**最后一句**：

> GitHub 是技术人的名片，但名片上不应该写着你的银行卡密码。花 10 分钟做好敏感信息治理，比花 10 小时处理安全事件要划算得多。

*文章基于 Haze AI Hub 项目真实脱敏过程整理。文中* `api-key` *为演示假值，真实项目请遵循”代码零密钥”原则。*
