### 登入数据库

**mysql -u -p -h<主机名>**

# 常用命令

### 创建数据库

**create database ;     创建一个名为的数据库**

### 查询数据库信息

**select database();                    查询当前所在数据库**
**show database;                        查看所有数据库**
**use ;            用于选择数据库**
**select version();               查询数据库版本**
**show status;                  显示一些特定资源的信息**
**show tables;                  显示数据库中的所有表**
**desc         查看表结构,显示字段名**
**SELECT ,,... FROM <table_name>;        从指定数据库中选取特定字段并返回选取结果**
**SELECT * FROM <table_name>;        从指定的数据库中选取所有字段信息并返回结果**

### 删除数据库信息

**drop database ;       删除数据库**

## table数据库表操作

#### 查看表结构

**desc ;            查看当前数据库中指定的表字段结构**
**describe ,      查看当前数据库中指定的表字段结构**

#### 建立表

**create table (**
					**<属性变量名/字段名> <数据类型> [完整约束条件]**
					**……**
					**……**
					**……**
**);**

#### 约束条件

**NOT NULL               指示某列不能存储 NULL 值**
**UNIQUE                   保证某列的每行必须有唯一的值**
**PRIMARY KEY         NOT NULL 和 UNIQUE 的结合。确保某列（或两个列多个列的结合）有唯一标识，有助于更容易更快速地找到表中的一个特定的记录**
FOREIGN KEY         保证一个表中的数据匹配另一个表中的值的参照完整性
CHECK                      保证列中的值符合指定的条件
**DEFAULT                   规定没有给列赋值时的默认值**
**INDEX                        用于快速访问数据库表中的数据**

#### 修改表

**alter table  rename ;           重命名表名**
**alter table  modify <属性变量名> <数据类型> [约束条件];       重新修改字段的数据类型**
**alter table  change   <新数据类型>;        修改字段名/重新修改字段的数据类型**
**alter table  drop <属性变量名>;           删除字段**

### 数据的增删改查

#### 增

**INSERT INTO  (,,,...) VALUES (,,,...);         在表中所有字段中添加数据**

**INSERT INTO  VALUES (,,,...);      在表中所有字段中添加数据，无需指定要插入数据的列名**

**ALTER TABLE <表名> ADD <新字段名> <数据类型> [约束条件] AFTER <已经存在的字段名>; 在中间位置添加一个新的字段**

**ALTER TABLE <表名> ADD <新字段名> <数据类型> [约束条件] FIRST;     在表字段的开头区域添加一个新的字段**

**ALTER TABLE <表名> ADD <新字段名><数据类型>[约束条件];    在表字段的末尾区域增加一个新字段**

#### 删

delete from <tablename></tablename> where <condition条件表达式>;         删除表中的指定行
**delete from ;          删除表中的全部数据，但不保留表结构**
**TRUNCATE TABLE ;      快速清空表中的所有数据，但保留表结构，并使任何自动增量值都重置为其初始值**
**ALTER TABLE <table_name> DROP COLUMN <column_name1>,DROP COLUMN <column_name2>,......;  删除表中指定的列**

#### 更新(改)

**update  set =,=,... WHERE <condition条件表达式>;                            提取那些满足指定条件的记录，更新这些条件记录**

#### 查(补)

###### 按找条件查询

**SELECT ,,... FROM  WHERE <condition条件表达式>;       查询选择满足指定条件的字段信息**

###### in查询

SELECT <column1></column1>,<column2></column2>,... FROM <tablename></tablename> WHERE <column></column> IN (<value1></value1>,<value2></value2>,...);                 查询选择满足指定字段中的指定值对应的字段信息

###### betwen and查询

SELECT `<column1></column1>,<column2></column2>`,... FROM `<tablename></tablename>` WHERE `<column></column>` BETWEEN `<value1></value1>` AND `<value2></value2>`;               选取`<column></column>`介于and连接的两个值之间的数据范围内的值，并返回查询字段结果

###### 带distinct关键字查询

**SELECT DISTINCT `<column1>`,... FROM `<tablename>`;        去除字段的重复值并返回对应的字段**

###### ==like查询(模糊查询)==

SELECT `<column1></column1>`,`<column2></column2>`,... FROM `<tablename></tablename>` WHERE <column></column> LIKE `<pattern搜索模式>`;                    在 WHERE 子句中进行模糊查询

- **通配符**
  - **%：匹配任意字符（包括零个字符）。**
  - **\_：匹配单个字符。**

###### and查询与or查询

SELECT `<column1></column1>`,`<column2></column2>`,... FROM `<tablename></tablename>` WHERE `<condition条件表达式>` and `<condition条件表达式>`;            多个条件满足时查询

SELECT `<column1></column1>`,`<column2></column2>`,... FROM `<tablename></tablename>` WHERE `<condition条件表达式>` or `<condition条件表达式>`;            单个条件满足时即可查询

##### 高阶查询使用

###### 聚合查询

-- COUNT: 统计记录数量
SELECT COUNT(<xxx></xxx>) FROM <tablename></tablename>;            返回指定列的值的数目（NULL 不计入）
SELECT COUNT(\*) FROM <tablename></tablename>;                 返回表中记录的所有字段的数目

-- SUM: 计算某列的总和
SELECT SUM(<xxx></xxx>) FROM <tablename></tablename>;             返回表中某列值的总和

-- AVG: 计算某列的平均值
SELECT AVG(<xxx></xxx>) FROM <tablename></tablename>;              返回表中某列值的平均值

-- MAX: 获取某列的最大值
SELECT MAX(<xxx></xxx>) FROM <tablename></tablename>;              返回表中某列值的最大值

-- MIN: 获取某列的最小值
SELECT MIN(<xxx></xxx>) FROM <tablename></tablename>;                返回表中某列值的最小值

###### 字符串函数

LOWER()                将字符串转换为小写
UPPER()                 将字符串转换为大写
REVERSE()            返回字符串值的相反顺序

###### 日期函数

NOW()                    返回当前日期和时间
DAY()                      返回月份中的一天（0-31）
MONTH()          	返回经过日期（1-12）的月份
YEAR()		      返回年份

###### 分组查询

> [!IMPORTANT]
> **SELECT ,... FROM  GROUP BY ;          依据GROUP BY后的字段名对结果进行分组查询并返回结果**

###### limit限制查询

select * from <tablename></tablename> limit `<num1></num1> <num2></num2>;`        查询id从<num1></num1>开始(不包括`<num1></num1>`)到`<num2></num2>`的字段

###### 为字段起别名

**select * from  as  from ;           给一个字段起另一个别名**
`select * from <oldfieldname></oldfieldname> as <newtablename></newtablename> where <newtablename></newtablename>.<fieldname></fieldname>;`               使用表的别名进行信息的查询

###### 为表起别名

**SELECT <column_name(s)> FROM <table_name> AS <alias_name>;      给一个表起别名**

- 其基本用法
  SELECT <s.stunum>,<s.stuname> FROM <table_name> AS <s></s>;

###### 子查询

###### where型子查询

**SELECT * FROM  WHERE  > (SELECT  FROM );**

###### from型子查询

**selece * from (select ,,…… from ) as  where ;**

###### exists型子查询

**SELECT ,,…… FROM  WHERE EXISTS (SELECT ,,…… FROM  WHERE );**            判断查询子句是否有记录，如果有一条或多条记录存在返回 True，否则返回 False

###### insert型子查询

INSERT INTO <premium_customers>
SELECT <xxx></xxx> FROM <xxx></xxx>
WHERE <xxx></xxx> IN (SELECT DISTINCT <cust_id> FROM <orders></orders> WHERE <xxx></xxx>);

###### 联合查询(合并查询)

- 合并两个查询结果（去重）

> [!IMPORTANT]
> **SELECT ,,…… FROM  WHERE  UNION SELECT ,,…… FROM  WHERE ;         合并两个或多个 SELECT 语句的结果集。从多个表中选择数据，并将结果集组合成一个结果集(union后可在加关键字all可将查询到的数据不去重)**

###### 连接查询

(**内连接（innner join，或称等值连接）**：返回两张表中匹配的记录
**左连接（left join）**：返回两张表匹配的记录，以及左表中多余的记录
**右连接（right join）**：返回两张表匹配的记录，以及右表中多余的记录
**全连接（full join）**：返回两张表匹配的记录，以及左右两表中各自多余的记录)

- 内连接
  **select * from  inner join  on .<class_id>=.;**     遍历左表数据，用左表数据的联属性去匹配右表的每一条数据。如果关联属性相同，则匹配到一条数据。

<img src="https://i-blog.csdnimg.cn/blog_migrate/8b776ec7cdb426a366a867bceb692de4.png" alt="内连接" style="zoom:50%;" />

- 左连接
  **select * from  left join  on .<class_id>=.;**        遍历左表数据，用左表数据的关联属性去匹配右表中的每一条数据。如果关联属性相同，则匹配到一条数据；如果左表数据没有匹配到右表的任何一条数据，则将右表数据的所有字段置为NULL，仍记作匹配到一条数据。

<img src="https://i-blog.csdnimg.cn/blog_migrate/f37e16eda09aadfb734c37a2e4d3525f.png" alt="左连接" style="zoom:50%;" />

- 右链接
  **select * from  right join  on .<class_id>=.;**    遍历右表数据，用右表数据的关联属性去匹配左表中的每一条数据。
  如果关联属性相同，则匹配到一条数据；
  如果右表数据没有匹配到左表的任何一条数据，则将左表数据的所有字段置为NULL，仍记作匹配到一条数据。

<img src="https://i-blog.csdnimg.cn/blog_migrate/8bf89575924d26b371185b32c1d3607a.png" alt="右链接" style="zoom:50%;" />

- 全连接
  **select * from  left join  on <stu.class_id>=<class.id>**
  **union**
  **select * from  right join  on <stu.class_id>=<class.id>;**
  先对左右表做左连接，再对左右表做右连接,两次连接的结果，去重后就是全连接的结果

<img src="https://i-blog.csdnimg.cn/blog_migrate/d76056aff35f442265cadd53762f0c70.png" alt="全连接" style="zoom:50%;" />

###### 索引

- **普通索引**
  **CREATE INDEX <索引名>**
  **ON  (<column1 [ASC|DESC]>, <column2 [ASC|DESC]>, ...);    对表的字段创建普通索引**
  (注：ASC和DESC（可选）: 用于指定索引的排序顺序。默认情况下，索引以升序（ASC）排序)
- **唯一索引**
  **CREATE UNIQUE INDEX **
  **ON  (<column1 [ASC|DESC]>, <column2 [ASC|DESC]>, ...);    对表的字段创建唯一索引，确保索引中的值是唯一的，不允许有重复值**
- 添加和删除索引

  - **索引删除**
    **DROP INDEX  ON ;**
    **ALTER TABLE  DROP INDEX ;             删除索引**
  - 在创建表的时候添加普通索引
    CREATE TABLE <students></students> (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    ......,
    INDEX <indexname></indexname> (<column1></column1>,....)
    );
  - 在创建表的时候添加唯一索引
    CREATE TABLE <tablename></tablename> (
    column1 data_type,
    column2 data_type,
    ...,
    CONSTRAINT <indexname></indexname> UNIQUE (<column1 [ASC|DESC]>, <column2 [ASC|DESC]>, ...)
    );
  - 修改表结构添加唯一索引

    - ALTER TABLE命令添加
      ALTER table <tablename></tablename>
      ADD CONSTRAINT <unique_constraint_name> UNIQUE (column1, column2, ...);
- 查看表索引
  **SHOW INDEX FROM ;           检查表现有索引**

###### 视图

- **创建视图**
  **CREATE VIEW  AS <条件>;           创建一个新视图**
- **替换现有视图**
  **CREATE OR REPLACE VIEW  AS <条件>;          替换一个视图**
  **(注：在CREATE VIEW语句中使用OR REPLACE子句时，如果该视图不存在，它将创建一个新视图，否则将替换现有视图)**
- **通过视图更新数据**
  **(注：**
  **通常，如果视图包含以下任何内容，则该视图不可更新：**

**DISTINCT，GROUP BY或HAVING子句。**

**聚合函数，例如AVG()，COUNT()，SUM()，MIN()，MAX()等等。**

**UNION，UNION ALL，CROSSJOIN，EXCEPT或INTERSECT运算符。**

**WHERE子句中的子查询引用FROM子句中的表。)**

**UPDATE  SET <salary = '6000'>**
**WHERE <条件>;          更新视图的数据**

- **删除视图**
  **DROP VIEW ;       删除一个视图**

###### 排序

**SELECT , ,**
**FROM <table_name>**
**ORDER BY <column1 [ASC | DESC]>, <column2 [ASC | DESC]>, ...;               ORDER BY 子句用于根据指定的列对结果集进行排序**

- **ASC: 按升序排序（默认）。**
- **DESC: 按降序排序。**

# 常用重要的SQL函数

![MySQL 常用函数](fbf6ea1f0572e3df2d622ca9fb50d50c_720.jpg)
![](f57d7cb9e4df313acd070bfe875271de_720.jpg)
![700](c90ec8d07374a7dac08766da24bb5456_720.jpg)
`if(<a>,<b>,<c>)`     if判断语句，`<a>为条件，<b>，<c>为执行语句;如果<a>语句为真则执行<b>语句，<a>语句为假则执行<c>语句`
`mid(<字段名>,<起始数>,<长度(可选参数)>)`     MID() 函数用于从文本字段中提取字符
`left(<字段名>,<长度>)`     从字符表达式最左边一个字符开始返回指定长度的字符


# 常用重要的SQL全局变量
![](Pasted%20image%2020260805165758.png)
\- 如果为空，则该变量没有作用。这不是一个安全的设置。
\- 如果设置为目录的名称，服务器将限制导入和导出操作，使其只能处理该目录中的文件。该目录必须存在；服务器不会创建它。
\- 如果设置为\`NULL\`，服务器将禁用导入和导出操作。

![](ae64b2fc10b01527e60525b736a58b64_720.jpg)
`@@long_query_time`     时间






























































































