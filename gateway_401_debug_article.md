# Spring Cloud Gateway 返回的 401，前端为什么”看不见”？

> ——一次 WebFlux 与 MVC 响应格式差异的排查记录

## 一、前言

最近在做一个校园教材共享平台的练手项目，技术栈是 Vue3 + Pinia + Spring Cloud Gateway + Sa-Token + Redis。登录鉴权模块跑通之后，我顺手做了一个”破坏性测试”：如果 Redis 里的会话 token 被删了，前端会立刻感知到并跳转登录页吗？

结果很诡异——**Redis 里的 token 已经没了，前端导航栏还显示着用户名，页面能正常跳转，甚至点击”申领教材”还提示”成功”**。但刷新个人中心，数据是空的；再一看数据库，根本没有生成申领记录。

前端明明”看着”是登录状态，后端却像没认这个用户。这篇文章记录了我从前端到网关、再到框架底层，逐层定位问题的全过程。

## 二、现象：Redis 删了 token，前端却进入”幽灵登录态”

我的登录流程是这样的：

1.  用户登录成功后，后端通过 Sa-Token 生成 token，写入 Redis；
2.  前端把 token 存到 `localStorage`，后续请求通过 Axios 拦截器带上 `Authorization` 请求头；
3.  所有请求先经过 Gateway 的 `SaReactorFilter` 做统一鉴权，再路由到下游服务。

为了验证”后端会话失效时系统的表现”，我登录后，直接去 Redis 里删掉了对应的 token：

`redis-cli`` del Authorization:login:token:5b7799af-f3a4-48d6-9702-b5593bb372d3`

然后回到浏览器，**不刷新页面**，直接点击导航栏的”我要捐赠”。

**预期**：页面被拦截，弹窗提示登录过期，跳转到登录页。  
**实际**：页面正常打开了，右上角还显示着用户名和退出按钮。点击”个人中心”也能进去，只是数据区域一片空白。

更离谱的是，在捐赠页面点击”确认申领”，前端竟然弹了”申领成功”的提示。但我去数据库一查，**根本没有这条记录**。

这说明请求确实走到了后端业务代码，但业务层似乎也没有拦住这个”非法用户”。

## 三、第一层排查：我删的 token，是不是浏览器正在用的那个？

我首先怀疑的是：**Redis 里是不是存了多个 token，我删错了？**

查看配置，发现 Sa-Token 开了这两个选项：

`sa-token``:`  
`  ``is-concurrent``:`` ``true``   # 允许同一账号多地登录`  
`  ``is-share``:`` ``false``       # 每次登录产生新 token，旧 token 仍有效`

这意味着每次登录都会在 Redis 里新增一个 token，旧的不会自动清理。我 `keys Authorization:login:token:*` 一查，里面有二十多个历史 token。我之前随手删的，很可能不是浏览器 `localStorage` 里存的那个。

为了确认，我精确比对了一次：

`// 浏览器 Console`  
`localStorage``.``getItem``(``'token'``)`  
`// '5b7799af-f3a4-48d6-9702-b5593bb372d3'`

`# Redis`  
`redis-cli`` get Authorization:login:token:5b7799af-f3a4-48d6-9702-b5593bb372d3`  
`# (nil)  ← 确实已经删了`

好，现在确认浏览器带的 token，在 Redis 里确实已经不存在了。但为什么还能”正常访问”？

## 四、第二层排查：用 fetch 绕过前端，直接看网关返回了什么

我怀疑前端 Axios 拦截器是不是哪里写漏了。于是直接用原生 `fetch` 发请求，绕过所有前端逻辑，看网关到底返回了什么：

`fetch``(``'/api/borrow/apply'``,`` {`  
`  ``method``:`` ``'POST'``,`  
`  ``headers``:`` {`  
`    ``'Content-Type'``:`` ``'application/json'``,`  
`    ``'Authorization'``:`` localStorage``.``getItem``(``'token'``)`  
`  }``,`  
`  ``body``:`` ``JSON``.``stringify``({``isbn``:``'9787111544937'``,`` ``requestId``:``'test123'``})`  
`})``.``then``(r ``=>`` {`  
`  ``console``.``log``(``'HTTP状态:'``,`` r``.``status``)``;`  
`  ``return`` r``.``text``()``;`  
`})``.``then``(text ``=>`` ``console``.``log``(``'响应体:'``,`` text))`

输出结果让我愣住了：

`HTTP状态: 200`  
`响应体: Result(code=401, message=登录已过期，请重新登录, data=null)`

**两个关键信息：**

1.  **HTTP 状态码是 200，不是 401。** 网关确实拦截了（返回了”登录已过期”），但它没有抛 HTTP 401，而是包装了一个 HTTP 200 + 业务错误。
2.  **响应体是** `Result(code=401...)`**，不是 JSON。** 这是 Java 对象的 `toString()` 输出，前端 `JSON.parse()` 会直接报错。

这就解释了为什么前端”看不见”这个错误：

-   Axios 拦截器里判断 `error.response?.status === 401`，但状态码是 200，不走这条分支；
-   响应体解析失败，`response.data` 拿不到，`res.code` 是 `undefined`；
-   拦截器里的 `isAuthError` 判断依赖 `code === 'A00010'` 或 `msg.includes('登录')`，但此时连 `msg` 都读不出来。

所以前端既不弹”登录过期”，也不跳转登录页，页面就停留在”看起来正常”的幽灵状态。

## 五、深入分析：为什么业务服务能返回 JSON，网关却不行？

问题定位到这里，我开始好奇：**为什么下游的 user-service、borrow-service 返回** `Result.fail(401, ...)` **时，前端能正常解析 JSON，而 Gateway 却返回了** `toString()` **字符串？**

### 5.1 业务服务（Spring MVC）的行为

在 borrow-service 里，我的 Controller 长这样：

`@PostMapping``(``"/apply"``)`  
`@SaCheckLogin`  
`public`` ``Result``<``BorrowRecordVO``>`` ``apply``(``@Valid`` ``@RequestBody`` ApplyDTO dto``)`` ``{`  
`    ``return`` ``Result``.``success``(``borrowService``.``apply``(``dto``));`  
`}`

当方法返回 `Result` 对象时，Spring MVC 会自动走 `HttpMessageConverter` 链，其中 `MappingJackson2HttpMessageConverter` 把对象序列化成 JSON 写入响应体。这是 Spring MVC 的”魔法”，开发者几乎感知不到。

### 5.2 网关（Spring WebFlux）的行为

但 Gateway 基于 **Spring WebFlux**（响应式栈），不是 MVC。它的鉴权过滤器配置如下：

`@Configuration`  
`public`` ``class`` SaTokenGatewayConfig ``{`  
`    ``@Bean`  
`    ``public`` SaReactorFilter ``saReactorFilter``()`` ``{`  
`        ``return`` ``new`` ``SaReactorFilter``()`  
`            ``.``addInclude``(``"/**"``)`  
`            ``.``setAuth``(``obj ``->`` StpUtil``.``checkLogin``())`  
`            ``.``setError``(``e ``->`` ``{`  
`                ``// 问题出在这里！`  
`                ``return`` ``Result``.``fail``(``401``,`` ``"登录已过期，请重新登录"``);`  
`            ``});`  
`    ``}`  
`}`

`SaReactorFilter.setError()` 的内部逻辑大致是：

`Object`` errorResult ``=`` errorHandler``.``run``();`  
`if`` ``(``errorResult ``instanceof`` ``String``)`` ``{`  
`    response``.``write``((``String``)`` errorResult``);`  
`}`` ``else`` ``{`  
`    response``.``write``(``String``.``valueOf``(``errorResult``));`` ``// ← 调用 toString()`  
`}`

**它不会调用 Jackson，不会走 MessageConverter，直接** `toString()`**。**

而我的 `Result` 类用了 Lombok 的 `@Data`，自动生成的 `toString()` 长这样：

`Result(code=401, message=登录已过期，请重新登录, data=null)`

这当然不是合法 JSON。前端 `axios` 默认按 JSON 解析，直接抛 `SyntaxError`，导致后续的错误处理逻辑全部失效。

### 5.3 本质差异

| 维度            | 业务服务（Spring MVC） | 网关（Spring Cloud Gateway） |
|-----------------|------------------------|------------------------------|
| 底层框架        | Servlet 栈             | WebFlux（响应式）            |
| 返回对象时      | 自动走 Jackson 序列化  | 不会自动序列化               |
| `Result` 的处理 | `{"code":401}`         | `Result(code=401...)`        |

**这不是框架 Bug，是架构差异。** 在 WebFlux 的过滤器里返回 Java 对象，不能期望它自动变成 JSON，必须手动处理。

## 六、前端视角：为什么”看着是登录态”本身就是一种陷阱

除了网关的问题，这次排查也让我重新审视了前端的状态管理。

### 6.1 Pinia Store 的恢复逻辑

`const`` token ``=`` ``ref``(localStorage``.``getItem``(``'token'``) ``||`` ``''``)`  
`const`` userInfo ``=`` ``ref``<``any``>``(``null``)`  
  
`const`` savedUser ``=`` localStorage``.``getItem``(``'userInfo'``)`  
`if`` (savedUser) {`  
`    ``try`` {`  
`        userInfo``.``value`` ``=`` ``JSON``.``parse``(savedUser)`  
`    } ``catch`` {}`  
`}`  
  
`const`` isLoggedIn ``=`` ``computed``(() ``=>`` ``!!``token``.``value``)`

页面刷新时，Pinia 直接从 `localStorage` 恢复 `token` 和 `userInfo`。只要 `token` 字符串不为空，`isLoggedIn` 就是 `true`，导航栏就显示头像和用户名。

**但** `localStorage` **里的 token 是否还有效，前端在初始化时是完全不知道的。**

### 6.2 路由守卫的判断逻辑

`router``.``beforeEach``((to``,`` ``from``,`` next) ``=>`` {`  
`    ``const`` userStore ``=`` ``useUserStore``()`  
`    ``const`` token ``=`` userStore``.``token`  
  
`    ``if`` (to``.``meta``.``requireAuth`` ``&&`` ``!``token) {`  
`        ``return`` ``next``(``'/login'``)`  
`    }`  
`    ``next``()`  
`})`

路由守卫只判断 `token` 这个字符串是否存在，**不校验它在后端是否还有效**。所以即使 Redis 里 token 已经没了，前端路由仍然放行，用户能正常进入 `/donate`、`/profile` 等需要登录的页面。

### 6.3 幽灵登录态的形成

综合以上三点，一个完整的”幽灵登录态”就形成了：

1.  **前端**：`localStorage` 有 token，`isLoggedIn = true`，显示登录态，路由放行；
2.  **网关**：校验 token 发现 Redis 里没有，应该返回 401，但返回了 `Result(...)` 字符串；
3.  **前端拦截器**：解析 JSON 失败，`code` 和 `msg` 都拿不到，`isAuthError` 判断为 `false`，不触发跳转；
4.  **用户感知**：页面能打开，看着是登录态，但发请求没数据，操作”成功”实际没生效。

## 七、修复方案

### 7.1 网关层：手动序列化为 JSON

在 `SaTokenGatewayConfig` 的 `setError` 里，不再直接返回 `Result` 对象，而是手动转 JSON：

`import`` ``com``.``fasterxml``.``jackson``.``databind``.``ObjectMapper``;`  
  
`@Configuration`  
`public`` ``class`` SaTokenGatewayConfig ``{`  
  
`    ``private`` ``static`` ``final`` ObjectMapper mapper ``=`` ``new`` ``ObjectMapper``();`  
  
`    ``@Bean`  
`    ``public`` SaReactorFilter ``saReactorFilter``()`` ``{`  
`        ``return`` ``new`` ``SaReactorFilter``()`  
`            ``.``addInclude``(``"/**"``)`  
`            ``.``setAuth``(``obj ``->`` StpUtil``.``checkLogin``())`  
`            ``.``setError``(``e ``->`` ``{`  
`                ``try`` ``{`  
`                    ``Result`` result ``=`` ``Result``.``fail``(``401``,`` ``"登录已过期，请重新登录"``);`  
`                    ``return`` mapper``.``writeValueAsString``(``result``);`  
`                ``}`` ``catch`` ``(``Exception`` ex``)`` ``{`  
`                    ``return`` ``"{"``code``":401,"``message``":"``登录已过期，请重新登录``"}"``;`  
`                ``}`  
`            ``});`  
`    ``}`  
`}`

如果担心网关模块没有引入 Jackson，也可以直接用字符串拼接，100% 安全：

`.``setError``(``e ``->`` ``{`  
`    ``return`` ``"{"``code``":401,"``message``":"``登录已过期，请重新登录``","``data``":null}"``;`  
`})`

### 7.2 前端层：增加兜底解析（可选）

在 `request.ts` 里增加对字符串响应的兼容：

`request``.``interceptors``.``response``.``use``(`  
`    (response``:`` AxiosResponse) ``=>`` {`  
`        ``const`` res ``=`` response``.``data`  
  
`        ``// 兜底：如果返回的是 Result(...) 字符串`  
`        ``if`` (``typeof`` res ``===`` ``'string'`` ``&&`` res``.``startsWith``(``'Result('``)) {`  
`            ``const`` codeMatch ``=`` res``.``match``(``/code=``(\d+)``/``)`  
`            ``if`` (codeMatch ``&&`` codeMatch[``1``] ``===`` ``'401'``) {`  
`                ElMessage``.``error``(``'登录已过期，请重新登录'``)`  
`                ``toLogin``()`  
`                ``return`` ``Promise``.``reject``(``new`` ``Error``(``'登录已过期'``))`  
`            }`  
`        }`  
  
`        ``// ... 原有逻辑`  
`    }`  
`)`

不过优先修网关，前端兜底只是保险。

### 7.3 清理历史 token

由于之前 `is-concurrent: true` + `is-share: false` 导致 Redis 里堆积了大量历史 token，测试完后顺手清理：

`redis-cli`` ``--scan`` ``--pattern`` ``"Authorization:login:*"`` ``|`` ``xargs`` redis-cli del`

## 八、总结与反思

这次排查让我对几个概念有了更深的理解：

**1. 前后端状态是分离的**  
前端 `localStorage` 的登录态只是”乐观假设”，真正的权威在后端 Redis。页面”显示已登录”不等于”后端认可你登录”。

**2. Gateway 与业务服务的错误格式必须统一**  
Spring MVC 和 Spring WebFlux 在响应处理上有本质差异。在 Gateway 过滤器里返回 Java 对象，不会自动序列化为 JSON，必须手动处理。

**3. 功能跑通 ≠ 链路可靠**  
登录功能能跑通只是 60 分。主动做异常测试（比如删 Redis token、断网、重启服务），验证系统在边界情况下的表现，才能发现问题。

**4. 配置项的副作用需要关注**  
`is-concurrent: true` + `is-share: false` 虽然实现了”多地登录”，但也导致 token 在 Redis 里无限堆积。理解每个配置项的含义，才能避免排查时”删错 token”的干扰。

> 最后，如果你也在用 Spring Cloud Gateway 做统一鉴权，建议检查一下你的 `setError` 返回的是不是 JSON。一个 `toString()` 的疏忽，可能让你的前端拦截器变成”瞎子”，用户就在”幽灵登录态”里不知不觉地操作了半天，最后发现数据根本没落库。
