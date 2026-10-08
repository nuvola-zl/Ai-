# MyBatis-Plus 踩坑记录：从 `BindingException` 到实体类规范

> 在学习 MyBatis-Plus 的过程中，我遇到了两个看似”莫名其妙”的报错。排查过程让我对框架的底层机制有了更深的理解，也意识到**实体类与数据库的一致性**有多么重要。

## 一、`selectById` 突然”消失”了

### 1.1 现象

项目启动正常，接口也能通，但调用某个 Mapper 的 `selectById` 时突然报错：

`org.apache.ibatis.binding.BindingException: `  
`Invalid bound statement (not found): `  
`com.example.demo.mapper.UserBorrowQuotaMapper.selectById`

更奇怪的是，同一个项目里另一个 `UserMapper.selectById` 却能正常工作。

### 1.2 初步排查

我第一反应是配置出了问题，于是依次检查了：

-   **pom 依赖**：确认引入的是 `mybatis-plus-boot3-starter`（Spring Boot 3 专用），版本 3.5.5，没问题。
-   **启动类扫描**：`@MapperScan("com.example.demo.mapper")` 路径正确。
-   **Mapper 接口**：确认继承了 `BaseMapper`，且加了 `@Mapper` 注解。
-   **XML 文件**：项目里没有写 XML，纯注解方式。

以上都没问题，但报错依然存在。

### 1.3 对比实验

既然 `UserMapper` 正常、`UserBorrowQuotaMapper` 异常，我决定**对比两个实体类的差异**。

**能正常工作的** `User` **实体类：**

`@Data`  
`@TableName``(``"sys_user"``)`  
`public`` ``class`` User ``{`  
`    ``@TableId``(``type ``=`` IdType``.``AUTO``)`  
`    ``private`` ``Long`` id``;`  
`    ``private`` ``String`` name``;`  
`    ``// ...`  
`}`

**报错的** `UserBorrowQuota` **实体类：**

`@Data`  
`@TableName``(``"user_borrow_quota"``)`  
`public`` ``class`` UserBorrowQuota ``{`  
`    ``private`` ``Long`` userId``;``  ``// ← 注意：没有 @TableId`  
`    ``private`` ``Integer`` currentBorrowCount``;`  
`    ``// ...`  
`}`

### 1.4 根因

MyBatis-Plus 的 `BaseMapper` 提供了通用的 CRUD 方法（如 `selectById`、`updateById`、`deleteById`）。但这些方法**依赖主键信息**来生成 `WHERE id = ?` 这样的 SQL。

框架默认认为主键字段叫 `id`。如果主键字段不叫 `id`（比如叫 `userId`），**必须显式标注** `@TableId`，否则 MyBatis-Plus 在启动时无法识别主键，也就**不会为这些方法注册 SQL 映射**。调用时自然报 `Invalid bound statement (not found)`。

### 1.5 解决

在主键字段上加上 `@TableId` 即可：

`@Data`  
`@TableName``(``"user_borrow_quota"``)`  
`public`` ``class`` UserBorrowQuota ``{`  
`    ``@TableId``  ``// ← 加上这个`  
`    ``private`` ``Long`` userId``;`  
`    ``private`` ``Integer`` currentBorrowCount``;`  
`    ``// ...`  
`}`

重启后，`selectById` 恢复正常。

### 1.6 原理补充

MyBatis-Plus 通过 `MybatisSqlSessionFactoryBean` 在应用启动时，将 `BaseMapper` 中的通用方法注入到 MyBatis 的 `Configuration` 中。注入过程中需要解析实体类的主键字段，如果找不到 `@TableId` 且字段名不是 `id`，注入就会跳过 `selectById` 等方法。这并非运行时错误，而是**启动时就没有注册**。

## 二、`Unknown column`：实体类比数据库”多嘴”

### 2.1 现象

定时任务执行查询时报错：

`org.springframework.jdbc.BadSqlGrammarException: `  
`### Error querying database.  `  
`Cause: java.sql.SQLSyntaxErrorException: `  
`Unknown column 'qr_code' in 'field list'`  
`### SQL: SELECT id, ..., qr_code, ... FROM borrow_record WHERE ...`

### 2.2 排查

查看数据库表结构：

`CREATE`` ``TABLE```  `borrow_record` ( ``  
``     `id` BIGINT  ```NOT`` ``NULL`` AUTO_INCREMENT,`  
``     `record_no`  ```VARCHAR``(``32``) ``NOT`` ``NULL``,`  
``     `status`  ```VARCHAR``(``16``) ``NOT`` ``NULL``,`  
`    ``-- ...`  
`    ``PRIMARY`` ``KEY```  (`id`) ``  
`);`

发现表里**根本没有** `qr_code` **字段**。

再查看实体类：

`@Data`  
`@TableName``(``"borrow_record"``)`  
`public`` ``class`` BorrowRecord ``{`  
`    ``@TableId``(``type ``=`` IdType``.``AUTO``)`  
`    ``private`` ``Long`` id``;`  
`    ``private`` ``String`` recordNo``;`  
`    ``private`` ``String`` status``;`  
`    ``private`` ``String`` qrCode``;``  ``// ← 数据库里没有这个字段！`  
`    ``// ...`  
`}`

### 2.3 根因

MyBatis-Plus 默认会**根据实体类的所有非 transient 字段生成 SQL**。当执行 `selectList` 时，它自动拼接了 `SELECT ... , qr_code , ...`，但数据库表里没有这列，MySQL 直接抛语法错误。

### 2.4 解决

如果该字段暂时不需要，直接从实体类中移除：

`@Data`  
`@TableName``(``"borrow_record"``)`  
`public`` ``class`` BorrowRecord ``{`  
`    ``@TableId``(``type ``=`` IdType``.``AUTO``)`  
`    ``private`` ``Long`` id``;`  
`    ``private`` ``String`` recordNo``;`  
`    ``private`` ``String`` status``;`  
`    ``// private String qrCode;  // ← 删掉或注释掉`  
`    ``// ...`  
`}`

如果业务需要这个字段，则必须先执行 `ALTER TABLE` 添加列，保持实体类与数据库表结构一致。

## 三、总结与建议

通过这次排查，我总结了以下几点：

| 问题                  | 现象                                     | 预防措施                                    |
|-----------------------|------------------------------------------|---------------------------------------------|
| 主键未标注 `@TableId` | `BindingException: selectById not found` | 主键字段不叫 `id` 时，**必须**加 `@TableId` |
| 实体类字段多于数据库  | `Unknown column 'xxx' in 'field list'`   | 修改实体类前，先同步数据库表结构            |

### 最佳实践

1.  **新建实体类时**，先对照数据库表，确认每个字段都存在，主键已标注 `@TableId`。

2.  **使用** `@TableField(exist = false)`：如果实体类需要临时字段（如 VO 属性），但不想映射到数据库，可以显式声明：

-   `@TableField``(``exist ``=`` ``false``)`  
    `private`` ``String`` tempRemark``;`

3.  **Code Review 时**，把”实体类与表结构一致性”作为检查项，避免低级错误。

## 结语

这两个报错表面上是”配置问题”和”SQL 语法问题”，但根因都在**实体类与框架约定的匹配**上。MyBatis-Plus 帮我们省了很多手写 SQL 的工作，但也要求我们更严格地遵守它的规范。理解框架的底层注入机制后，排查思路就会清晰很多。

希望这篇记录对你有帮助，也欢迎交流指正！
