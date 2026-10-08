MyBatis-Plus 的"静默陷阱"：当 updateById 遇到 null

项目背景：Haze-AI-Hub 设备申领系统，Spring Boot + MyBatis-Plus + PostgreSQL。在联调测试"取消申领"功能时，发现了一个没有报错、没有异常、但数据就是不对的诡异 Bug。

一、现象：取消成功了，但数据没清干净

测试步骤很简单：

用户申请一台 macbook_pro

DAG 跑完，状态变为"待领取"，allocated_device_id = 13

用户点击"取消"，前端提示"取消成功"

查数据库验证：

sql

SELECT request_no, status, allocated_device_id FROM admin_device_request WHERE request_no = 'DEV-20260806-004';

预期：status = 6（已取消），allocated_device_id = NULL

实际：status = 6 ✅，但 allocated_device_id = 13 ❌

设备明明已经回库了（status = 1），但申领单上还"挂"着设备 ID。这意味着补偿任务每 10 分钟扫一次，会发现这条"已取消但还有设备"的记录，重复释放库存，导致库存虚增。

二、排查：日志正常，代码也"正常"

先看当时的代码：

java

// DeviceUserController.java — 取消"待领取"的申领@Transactionalpublic void cancelAllocated(AdminDeviceRequest request) {

// 1. 回滚设备

if (request.getAllocatedDeviceId() != null) {

AdminDeviceDetail device = detailMapper.selectById(request.getAllocatedDeviceId());

if (device != null) {

device.setStatus(1); // 改回在库

device.setAssignedUserId(null); // 清空用户

device.setAssignedAt(null); // 清空时间

detailMapper.updateById(device); // ← 这里

}

}

// 2. 回滚库存...（省略）

// 3. 改申领单状态

request.setStatus(6); // 已取消

request.setAllocatedDeviceId(null); // 清空设备ID

requestMapper.updateById(request); // ← 这里}

诡异之处：

没有异常

日志打印正常

updateById 返回了 1（更新成功）

但数据库里的 allocated_device_id 和 assigned_user_id 就是没变

我在数据库里反复确认，甚至怀疑是事务没提交。最后把 SQL 日志打开，终于发现了真相：

sql

-- 实际生成的 SQL（设备表）UPDATE admin_device_detail SET status = 1 WHERE id = 13-- assigned_user_id 和 assigned_at 根本没出现！

-- 实际生成的 SQL（申领单）UPDATE admin_device_request SET status = 6 WHERE id = 6-- allocated_device_id 根本没出现！

问题定位：setXxx(null) 被 MyBatis-Plus 静默忽略了。

三、根因：update-strategy: not_null

查 application.yaml：

yaml

mybatis-plus:

global-config:

db-config:

update-strategy: not_null \# ← 罪魁祸首

MyBatis-Plus 的 updateStrategy 有三种模式：

表格

| 策略             | 行为                       | 适用场景              |
|------------------|----------------------------|-----------------------|
| IGNORED          | 所有字段都更新，包括 null  | 需要强制清空的场景    |
| NOT_NULL（默认） | 只更新非 null 字段         | 大多数 CRUD，防止误清 |
| NOT_EMPTY        | 只更新非 null 且非空字符串 | 字符串字段保护        |

NOT_NULL 的设计初衷是好的——防止你不小心把字段更新成 null。但代价是：当你真的想清空字段时，MP 不给你机会。

更坑的是，这个过程完全静默。没有异常、没有警告、updateById 还返回 1（影响行数）。只有打开 SQL 日志或查数据库才能发现。

四、修复：LambdaUpdateWrapper 强制写入

MP 提供了绕过策略的方式：LambdaUpdateWrapper.set() 不受 update-strategy 限制。

修复后的代码：

java

// 设备表：强制清空 assigned_user_id / assigned_atLambdaUpdateWrapper\<AdminDeviceDetail\> deviceWrapper = new LambdaUpdateWrapper\<\>();

deviceWrapper.eq(AdminDeviceDetail::getId, request.getAllocatedDeviceId())

.set(AdminDeviceDetail::getStatus, 1)

.set(AdminDeviceDetail::getAssignedUserId, null) // ← 强制写入 null

.set(AdminDeviceDetail::getAssignedAt, null); // ← 强制写入 null

detailMapper.update(null, deviceWrapper);

// 申领单：强制清空 allocated_device_idLambdaUpdateWrapper\<AdminDeviceRequest\> requestWrapper = new LambdaUpdateWrapper\<\>();

requestWrapper.eq(AdminDeviceRequest::getId, request.getId())

.set(AdminDeviceRequest::getStatus, 6)

.set(AdminDeviceRequest::getAllocatedDeviceId, null); // ← 强制写入 null

requestMapper.update(null, requestWrapper);

生成的 SQL：

sql

UPDATE admin_device_detail SET status = 1, assigned_user_id = NULL, assigned_at = NULL WHERE id = 13

UPDATE admin_device_request SET status = 6, allocated_device_id = NULL WHERE id = 6

完美。

\*\*另一种方案：字段级注解（适合个别字段需要强制更新 null 的场景）\*\*

如果你只是某个特定字段需要允许更新为 null，不想改所有调用处的代码，可以在实体类上加注解：

\`\`\`java

public class AdminDeviceRequest {

// 其他字段保持默认 NOT_NULL 策略

@TableField(updateStrategy = FieldStrategy.IGNORED)

private Long allocatedDeviceId; // 这个字段允许被更新为 null

}

这样 updateById(entity) 就会正常更新 allocatedDeviceId = null，不需要改用 LambdaUpdateWrapper。

怎么选？

如果项目里大量字段都需要强制更新 null → 改全局策略为 IGNORED，或统一用 LambdaUpdateWrapper

如果只有个别字段需要 → 用 @TableField(updateStrategy = FieldStrategy.IGNORED) 更轻量

五、举一反三：还有哪些地方会中招？

在这个项目里，我扫了一遍所有"清空字段"的场景，发现 4 个文件都有同样问题：

表格

| 文件                                   | 场景                 | 修复方式                      |
|----------------------------------------|----------------------|-------------------------------|
| DeviceUserController.cancelAllocated() | 用户取消申领         | LambdaUpdateWrapper.set(null) |
| AdminManageController.confirmReturn()  | 管理员确认归还入库   | LambdaUpdateWrapper.set(null) |
| DeviceCompensationService.compensate() | 定时补偿任务释放资源 | LambdaUpdateWrapper.set(null) |
| DeviceTimeoutJob.autoRecycle()         | 超时自动回收设备     | LambdaUpdateWrapper.set(null) |

统一修复原则：凡是业务上需要"清空"某个字段的，一律用 LambdaUpdateWrapper，不再用 updateById(entity)。

六、总结

这个 Bug 给我两个教训：

ORM 框架的"自动化"是有代价的。MyBatis-Plus 的 not_null 策略帮你防了误操作，但也挡住了你的正常操作。用之前必须理解它的约定。

没有异常不等于没有 Bug。这个问题如果在生产环境，补偿任务会反复释放库存，导致库存越滚越多，可能几天后才会被发现。联调阶段把 SQL 日志打开，多查数据库，是避免这类静默 Bug 的最好办法。

如果你也在用 MyBatis-Plus，建议检查一下项目里是否有类似的"清空字段"操作。一个配置项，可能藏着让你加班的坑。
