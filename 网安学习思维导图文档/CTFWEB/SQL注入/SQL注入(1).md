## 补：MySQL中一个重要的数据库(information_schema)
information_schema数据库保留了mysql服务器所有数据库的信息；==如：数据库名，数据库的表，表栏的数据类型与访问权限等信息==。

**information_schema的表schemata中的列schema_name字段记录了所有数据库的名字**
**information_schema的表tables中的列table_schema字段记录了所有数据库的名字**
**information_schema的表tables中的列table_name字段记录了所有数据库的表的名字**
**information_schema的表columns中的列table_schema字段记录了所有数据库的表的名字**
**information_schema的表columns中的列table_name字段记录了所有数据库的表的名字**
**information_schema的表columns中的列column_name字段记录了所有数据库的表中的字段的名字**

### information_schema补充内容
![](0ba1d5b9ceac38a9fafa344de50a009c_720.jpg)

## SQL注入原理介绍
==程序把用户输入的数据直接拼接进 SQL 语句，导致数据库把原本应当作为“普通数据”的输入，当成了 SQL 指令的一部分执行。==

### 在webSQL注入中常用的几种SQL注释方式

`#`  --> `#`的URL编码为`%23`(在注入中，可能使用`#`号会收到URL编码的影响，导致注释失效，可以尝试使用将`#`转换为URL编码`%23`来注释)
`--`
`--+`  --> ` `(空格)可以使用`+`号来表示(同样是因为URL编码的原因)

## SQL常见注入的方法分类

简单注入(万能密码、引导闭合等)
布尔盲注——回显判断
时间注入——无回显
联合注入——字段回显位
报错注入——有报错信息回显
堆叠注入——执行任意语句

## SQL输入常用函数
[MySQL数据库常用命令](MySQL数据库常用命令.md)


## SQL注入类型
### 字符/数字型注入
SQL注入中可==将SQL语句注入简单类型分为两种：数字型注入与字符型注入==
![](0f1cb2e69412c1623bea10403af126af_720.jpg)
字符型传参可能还会遇见如`id=(' <str> ')` 样式的设计规则

## 联合注入
使用`union`关键字链接两张表进行查询；并使用子查询中的查询语句执行payload
![](26365292dbbefffcd5a757c01a9192c4_720.jpg)


## 报错注入
报错错误是通过==特殊的函数错误使用==使其输出错误结果来获取信息的

可以利用报错注入的前提：==页面有错误信息显示出来==

### 报错注入类型
1. ==数据类型溢出==
2. ==函数参数格式错误==
3. ==主键/字段重复==

### 大整数溢出报错注入
将需要查询的数据放入一个数值表达式中，再故意让该表达式超过 `BIGINT` 或 `DOUBLE` 的表示范围，使 MySQL 在错误消息中输出相关表达式，从而形成信息回显。

#### 常见形式可以分成两类：

1. **BIGINT 整数运算溢出**
    
2. **`EXP()` 指数运算溢出**
`EXP(<num>)` 函数功能为为自然常数 `e` 的 `X` 次幂

![](0d66dfc310dd169d222f310ed66ac821_720.jpg)

创造该错误的基本语句格式:`select EXP(~(<标量子查询语句>))`

==其中“标量子查询”表示只返回**一列**，**最多一行**信息。==
### 补：XML文档与XPath
XML，全称 **Extensible Markup Language**，中文称为**可扩展标记语言**。它是一种使用标签描述和组织数据的文本格式，主要用于**数据存储、配置管理以及不同系统之间的数据交换**。

![](f2c7cd3464c7de22c57288b4477b4b43_720.jpg)

### XPath报错注入

#### Extractvalue函数介绍
`EXTRACTVALUE()` ==是 MySQL 中用来从 XML 数据里提取内容的函数==。它接收两个参数：目标 XML 文档和 XPath 路径表达式。因为输入非法的 XPath 格式会引发报错并带出查询结果，它常被用在安全测试的报错注入中。

根据 XPath 规则，从 XML 中抓取对应标签内部的文本值

- 语法：
	`EXTRACTVALUE(xml_target, xpath_expression)`

**参数说明**：
- `xml_target`：字符串或 XML 文档对象。
- `xpath_expression`：用来定位节点的 XPath 路径。

#### XPath报错注入原理
SELECT extractvalue('', '/a/b');    这是正确的语句，第二个参数 xml中的位置是我们编写注入语句的地方，xml文档中查找字符位置是用/xxx/xxx/xxx/…这种格式进行查找的，如果我们写入其他格式，就会报错，并且会返回我们写入的非法格式内容，而这个非法的内容就是我们要爆破的数据。

![](a8698941d69219e8874f6cdf83933c22_720.jpg)

创造该错误的基本语句格式：`extractvalue(1,concat(0x7e,substr((<注入代码>),1,31)))`


### 主键重复报错注入

#### Rand()函数与floor()函数介绍
- rand()
`RAND()` 函数==用来生成一个随机小数，范围一般是 0 到 1 之间（包含 0，但不包含 1）==。它在各个数据库（如 MySQL、SQL Server）中广泛用于数据随机排序、抽样或生成随机测试数据，返回一个浮点小数。

- 语法：
	`RAND() / RAND(seed);`

通过对 `RAND()` 的结果进行数学运算，可以得到你想要的范围：
- 例：
	- **0 到 10 之间的小数**：`SELECT RAND() * 10;`
	- **0 到 10 之间的随机整数**：`SELECT FLOOR(RAND() * 10);` （向下取整）

- Floor()
`FLOOR()` 函数==用于**向下取整**（即返回小于或等于指定数值的最大整数）==。无论是正数还是负数，它都会将数值向数轴的左侧（更小的方向）舍入。

- 语法：
	`FLOOR(number)`

(略：作用与大多数高级编程语言中的floor函数相似)


#### 主键重复报错注入原理
**利用 `GROUP BY + RAND() + 聚合函数`，让数据库内部临时表产生**重复唯一键，从而通过 `Duplicate entry` 错误回显查询结果
\- rang(0)*2的固定性
\- rand函数在查询时会先执行一次，插入时会执行一次
\- group by比较时先从虚拟表中查询，后插入新表。新表中的group by后面的字段作为主键

![](99b3e73c65e498249cc20de8bb0522ac_720.jpg)
创造该错误的基本语句格式：`select count(*),(concat(floor(rand(0)*2),(<注入语句>)))x from users group by x;#`


[SQL注入(2)](SQL注入(2).md)

































































































