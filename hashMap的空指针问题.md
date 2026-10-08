DAG 并行节点的"并发之殇"：一个 HashMap 引发的空指针

项目背景：Haze-AI-Hub 设备申领系统，自研轻量级 DAG 工作流引擎。在联调测试时，configPermission 节点偶尔抛 NPE，重启后又好了，极其诡异。

一、现象：偶发 NPE，重启就"自愈"

测试"有库存申领"流程时，大部分情况下 DAG 能正常跑完：

plain

checkInventory → branchByStock → allocateDevice → installSystem + configPermission → notifyPickup

但偶尔会出现这种情况：

前端显示"提交成功"

等了几秒查进度，发现状态卡住了

查后端日志：

plain

\[ERROR\] 节点 configPermission 执行异常: null

\[ERROR\] 节点 configPermission 执行失败: null

java.lang.NullPointerException

at top.hazenix.hazeaihub.common.context.DagContext.toJson(DagContext.java:46)

最诡异的是：同样的操作，再试一次就正常了。没有固定复现路径，也没有修改任何代码。

这让我一度怀疑是刚才修的 MyBatis-Plus null 问题导致的。但仔细一想——取消/归还的代码在正常的申领流程里根本不会执行，路径完全不同。

二、排查：从日志里找"时间差"

打开 sys_dag_execution 表，对比一次"成功"和一次"失败"的执行记录：

成功的 DAG：

表格

| id  | current_node     | status | created_at   |
|-----|------------------|--------|--------------|
| 14  | checkInventory   | 2      | 16:21:46.867 |
| 15  | branchByStock    | 2      | 16:21:46.878 |
| 16  | allocateDevice   | 2      | 16:21:46.885 |
| 17  | installSystem    | 1      | 16:21:46.909 |
| 18  | configPermission | 1      | 16:21:46.913 |
| 19  | installSystem    | 2      | 16:21:47.428 |
| 20  | configPermission | 2      | 16:21:47.427 |
| 21  | notifyPickup     | 2      | 16:21:47.438 |

失败的 DAG：

表格

| id  | current_node     | status | created_at   |
|-----|------------------|--------|--------------|
| 69  | checkInventory   | 2      | 18:34:50.752 |
| 70  | branchByStock    | 2      | 18:34:50.761 |
| 71  | allocateDevice   | 2      | 18:34:50.766 |
| 72  | installSystem    | 1      | 18:34:50.783 |
| 73  | configPermission | 1      | 18:34:50.784 |
| 74  | installSystem    | 2      | 18:34:51.290 |
| 75  | configPermission | 3      | 18:34:51.291 |

注意到一个细节：installSystem 和 configPermission 的开始时间戳几乎一样（相差 1ms），说明它们是并行执行的。

再看报错堆栈，定位到 DagContext.toJson()：

java

// DagContext.javapublic String toJson() {

Map\<String, Object\> snapshot = new HashMap\<\>();

// ...

for (Map.Entry\<String, NodeResult\> entry : results.entrySet()) {

NodeResult nodeResult = entry.getValue();

// ← NPE 在这里！nodeResult 为 null

r.put("success", nodeResult.isSuccess());

}}

问题浮现：toJson() 在遍历 results 这个 HashMap，而另一个线程正在往里面 put。

三、根因：HashMap 不是线程安全的

我们的 DAG 引擎里，installSystem 和 configPermission 没有依赖关系，所以被丢进了线程池并行执行：

java

// DagEngine.java 伪代码CompletableFuture\<NodeResult\> futureA = CompletableFuture.supplyAsync(

() -\> installSystemNode.execute(context), executor);CompletableFuture\<NodeResult\> futureB = CompletableFuture.supplyAsync(

() -\> configPermissionNode.execute(context), executor);

两个线程共享同一个 DagContext 对象。每个节点执行完后，都会做两件事：

context.putResult(nodeId, result) —— 往 HashMap 里写

saveExecution(context) —— 把 context 序列化成 JSON 存数据库

putResult 和 toJson 是并发执行的：

plain

线程A（installSystem） 线程B（configPermission）

│ │

▼ ▼

putResult("installSystem", ...) putResult("configPermission", ...)

│ │

▼ ▼

toJson() 遍历 results ───────────────┘

│

▼

遍历到 entry("configPermission", ???)

此时线程B的 put 还没完全写完！

entry.getValue() 返回 null → NPE

这就是经典的 HashMap 并发读写问题。

更危险的是，HashMap 在并发 put 时发生 resize，还可能造成链表成环，导致 CPU 100% 死循环。这是 JDK7 时代的经典面试题，没想到在自研 DAG 引擎里踩到了。

四、修复：两步兜底

第一步：HashMap → ConcurrentHashMap

java

// 修复前public class DagContext {

private Map\<String, NodeResult\> results;

public DagContext(...) {

this.results = new HashMap\<\>(); // ← 线程不安全

}}

// 修复后public class DagContext {

private Map\<String, NodeResult\> results;

public DagContext(...) {

this.results = new ConcurrentHashMap\<\>(); // ← 线程安全

}}

ConcurrentHashMap 在 JDK8+ 使用 CAS + synchronized，保证并发 put 安全，而且遍历不会抛 ConcurrentModificationException。

第二步：toJson() 加 null 防御

java

public String toJson() {

Map\<String, Object\> snapshot = new HashMap\<\>();

// ...

for (Map.Entry\<String, NodeResult\> entry : results.entrySet()) {

NodeResult nodeResult = entry.getValue();

if (nodeResult == null) continue; // ← 双重保险

Map\<String, Object\> r = new HashMap\<\>();

r.put("success", nodeResult.isSuccess());

r.put("data", nodeResult.getData());

r.put("skipped", nodeResult.isSkipped());

snapshot.put(entry.getKey(), r);

}

// ...}

即使极端情况下读到 null，也能优雅跳过，不会阻断整个 DAG。

\*\*额外注意：CompletableFuture 的线程池\*\*

上面伪代码用了 \`CompletableFuture.supplyAsync()\`，它默认用的是 \`ForkJoinPool.commonPool()\`。DAG 节点如果都是 IO 操作（查库、调接口），common pool 的线程数等于 CPU 核数，很容易被占满，导致其他用了并行流的地方阻塞。

建议给 DAG 引擎配一个独立的线程池：

\`\`\`java

// DagEngine.java

private final ThreadPoolExecutor dagExecutor = new ThreadPoolExecutor(

4, // 核心线程数

8, // 最大线程数

60L, TimeUnit.SECONDS, // 空闲线程存活时间

new LinkedBlockingQueue\<\>(100), // 队列容量

new ThreadFactoryBuilder().setNameFormat("dag-pool-%d").build(),

new ThreadPoolExecutor.CallerRunsPolicy() // 队列满了，主线程自己执行

);

// 提交并行节点时用自定义线程池

CompletableFuture\<NodeResult\> futureA = CompletableFuture.supplyAsync(

() -\> installSystemNode.execute(context), dagExecutor);

CompletableFuture\<NodeResult\> futureB = CompletableFuture.supplyAsync(

() -\> configPermissionNode.execute(context), dagExecutor);

这样 DAG 的执行不会污染业务线程池，也方便单独监控和调优。

五、总结：自研引擎里的"共享可变状态"

这个 Bug 的核心教训是：只要多个线程共享可变状态，就必须显式保证线程安全。

在 DAG 引擎这种"并行节点"场景下，很容易忽略这一点——因为每个节点看起来是独立的，但它们共享的 DagContext 是一个全局可变容器。

表格

| 场景              | 是否线程安全 | 建议                                            |
|-------------------|--------------|-------------------------------------------------|
| 节点间无共享数据  | ✅ 安全      | 放心并行                                        |
| 共享只读数据      | ✅ 安全      | 放心并行                                        |
| 共享可变 Map/List | ❌ 危险      | 必须用 ConcurrentHashMap / CopyOnWriteArrayList |
| 共享可变对象字段  | ❌ 危险      | 加锁或原子类                                    |

另外，这个 Bug 也说明了"偶发"问题往往比"必现"问题更难排查。因为它依赖线程调度的时间窗口，本地调试时 CPU 负载低，可能 100 次都复现不了；到了测试环境，服务负载一高，竞态条件就出现了。

排查这类问题的最好方法：抓日志里的时间戳，看并发操作的时序关系。

如果你也在写异步编排、工作流引擎，或者用了 CompletableFuture 并行执行任务，建议检查一下共享的上下文对象。一个 new HashMap\<\>() 可能藏着让你半夜起床重启服务的坑。
