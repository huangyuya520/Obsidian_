
# 利用Soap协议触发SSRF攻击

## 补：Soap协议简介
SOAP（简单对象访问协议）是==一种基于 XML 的标准通信协议，用于让运行在不同操作系统上、使用不同编程语言的应用程序通过网络安全地交换结构化数据==



==SOAP 的核心特点==

- **跨平台与跨语言**：因为使用通用的 XML 格式，Windows 和 Linux 系统之间、Java 和 C# 程序之间可以无缝沟通。

- **高度标准化与安全性**：自带严格的规则和规范，并内置了对加密和数字签名的强大支持（WS-Security），适合金融和企业级应用。

- **独立于传输层**：虽然通常基于 [HTTP 协议](https://www.ibm.com/docs/zh/rsas/7.5.0?topic=standards-soap)传输，但它也支持 SMTP 等其他协议




==SOAP 消息的结构==

一个标准的 SOAP 消息是一个 XML 文档，主要包含四个部分：

- **Envelope（信封）**：必需的根元素，标明该 XML 文档是一个 SOAP 消息。
- **Header（头部）**：可选部分，包含身份验证、安全或路由等附加信息。
- **Body（主体）**：必需部分，包含发送方向接收方请求或响应的具体核心数据。
- **Fault（错误）**：可选部分，用来传输处理消息时发生的错误信息



例：
![](e0782594d760051c12cf6eb4ceed697b.jpeg)





## 补：php SoapClient原生类简介
`SoapClient` 是一个用于访问 SOAP 服务的内置原生类，可作为客户端与 SOAP 1.1 或 SOAP 1.2 服务器进行通信



==基本概念与工作模式==

- **协议基础**：底层使用 HTTP 作为通讯协议，以 XML 格式传输数据。 [[1](https://www.cnblogs.com/hetianlab/p/15194462.html)]

- **两种模式**：
    - **WSDL 模式**：通过传入 WSDL 文件的 URL 地址来自动构建服务方法。
    - **非 WSDL (non-WSDL) 模式**：不使用 WSDL 文件，直接在构造函数中指定目标 URL 和命名空间等参数



==核心特点==

- **远程过程调用 (RPC)**：允许 PHP 程序像调用本地方法一样调用远程 Web 服务。

- **魔术方法**：内置 `__call` 方法，当调用类中不存在的方法时，会自动转为对远程 SOAP 服务的请求




### 对SoapClient原生类触发SSRF攻击进行底层代码分析(C语言审计)
![](aaafc9559e2015caf2a396811731cc40_720.jpg)


![](3ea57d1d4e7ebf3e3870dcc35fc3a4b5_720.jpg)


![](fecfa2e55685c76ae4dfb35fe0ceb814_720.jpg)



![](179eb114b154443d45819610797ecb22_720.jpg)


![](8913f8245f156ef45bedcae1eaa6b88c_720.jpg)


![](cf843007c347508f49fd5a5194c72b00_720.jpg)


![](408a2b5e438ccb4c19a0c11a07e65728_720.jpg)






## 利用php Soap类触发SSRF攻击场景

### 使用Soap类实例化后直接调用不存在的函数进行SSRF攻击
在 PHP 中，当==以非 WSDL 模式实例化内置类 `SoapClient` 并直接调用一个不存在的方法时，底层会自动触发其内置的 `__call` 魔术方法。该方法会将函数名及参数封装为标准的 SOAP 协议格式，并向配置中指定的 `location` 地址发送一条 HTTP POST 请求，从而实现 **SSRF（服务端请求伪造）**==


1. ==核心触发机制与原理==

`SoapClient` 是 PHP 内置用于处理 SOAP 协议的类（依赖 `php-soap` 扩展）。

1. **非 WSDL 模式实例化**：构造函数第一个参数 `$wsdl` 传入 `null` 时，必须在第二个参数 `$options` 数组中提供 `location`（目标请求地址）和 `uri`（命名空间）。
    
2. **`__call` 拦截机制**：当调用实例上未定义的方法（如 `$client->notExistFunc()`）时，PHP 触发 `__call`。
    
3. **发起 HTTP POST 请求**：`__call` 组织 SOAP XML 请求体，向 `location` 对应的 URL 发起 HTTP POST 请求




基础触发代码示例

例：
![](Pasted%20image%2020260905111736.png)





生成的原始 HTTP 数据包

当上述代码执行时，服务器对外发出的原始 HTTP 请求如下：

例：
![](Pasted%20image%2020260905111814.png)







2. ==攻击进阶：利用 CRLF 注入伪造 HTTP 请求==

默认情况下，`SoapClient` 只能发起固定格式的 XML POST 请求。但通过在 `$options['user_agent']` 参数中注入 `\r\n`（CRLF），攻击者可以截断原 HTTP 头部，注入自定义 Header，甚至控制 HTTP POST Body。

攻击 Redis / 内部 RESTful API 示例

通过注入 `Cookie`、覆盖 `Content-Type` 和 `Content-Length`，可直接伪造表单或协议指令

例：
![](Pasted%20image%2020260905111922.png)



3. ==常见实战场景与利用链==

|**场景分类**|**出现形式**|**触发逻辑**|
|---|---|---|
|**直接调用**|业务中存在动态调用外部微服务的逻辑|开发者通过 `$client->$method()` 调用可控方法名，攻击者传入未定义方法触发 `SoapClient::__call`。|
|**反序列化利用链**|反序列化点（`unserialize`）结合 POP 链|某些类的 `__destruct()` 或其他魔术方法中包含 `$this->obj->func()` 调用，反序列化时将 `$this->obj` 赋值为 `SoapClient` 实例，在对象销毁时自动触发调用。|




4. ==防护与排查策略==

- **关闭非必要扩展**：在不需要 SOAP 通信的环境下，移除或禁用 `php-soap` 扩展。
    
- **URL 校验与白名单**：若必须动态实例化 `SoapClient`，严格比对 `location` 的域名与 IP，拒绝私有 IP 地址段（如 `127.0.0.1`、`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`）及非标端口。
    
- **禁止控制 Options 头部**：限制外部直接传递给 `user_agent`、`headers` 等参数的内容，严格过滤 `\r` (`%0d`) 和 `\n` (`%0a`) 字符，防止 CRLF 头部注入





例：
![](db5dde6a63fff66fd71ca7ead198a684_720.jpg)










### 使用Soap类任意类实例化进行SSRF攻击
在 PHP 应用开发中，==若存在**类名与构造函数参数由外部输入控制**的代码缺陷（即“任意类实例化”），攻击者可以指定实例化 PHP 内置的 `SoapClient` 类，从而直接触发 SSRF（服务端请求伪造）==


1. ==核心触发原理==


(1) 漏洞代码场景

动态实例化模式通常形如：

例：
![](Pasted%20image%2020260905114411.png)



(2) `SoapClient` 构造函数机制

PHP 的 `SoapClient` 核心功能是处理 SOAP Web 服务的客户端请求。其构造函数签名如下：

例：
```
public SoapClient::SoapClient(?string $wsdl, array $options = [])
```


- **WSDL 模式**：当传入第一个参数 `$wsdl` 为非空字符串时，`SoapClient` 会将其视为 WSDL 规范文件的路径或 URL。
    
- **即时网络外连**：在执行 `new SoapClient("http://target-host/...")` 的瞬间，PHP 底层会**立即发起一条 HTTP GET 请求**去下载并解析该 WSDL 文件。
    
- **异常不影响网络请求**：虽然目标端返回的内容通常不是合法的 XML（导致随后抛出 `SoapFault` 异常），但在抛出异常之前，**HTTP 请求已经在网络层实际发出并完成了通信**





2. ==攻击场景与利用方式==

场景 A：仅能控制单个参数（基础 HTTP GET SSRF）

**代码原型：**

例：
```
$obj = new $_GET['class']($_GET['param']);
```



**攻击请求：**

例：
```
GET /vuln.php?class=SoapClient&param=http://192.168.1.100:8080/api/internal HTTP/1.1
Host: example.com
```


**可实现的目标：**

1. **内网端口探测与存活检测**：根据响应时间、报错信息（连接被拒 `Connection refused` vs 响应超时 `timed out` vs 返回内容非 XML 错误）判断内网服务及端口开放状态。
    
2. **访问云服务元数据**：在云原生环境中请求元数据 API（如 AWS/GCP 的 `[http://169.254.169.254/latest/meta-data/](http://169.254.169.254/latest/meta-data/)`），窃取临时凭证或 IAM Role。
    
3. **触发无鉴权内部接口**：向无需验证且可通过 GET 请求触发动作的内网 REST API、Webhook 发送请求





场景 B：可控制两个参数（CRLF 注入与 HTTP 头走私）

如果业务代码允许传入数组作为第二个参数（或可控制构造函数的多参数）：

例：
![](Pasted%20image%2020260905114641.png)


`SoapClient` 的第二个参数 `$options` 包含众多配置项，其中 `user_agent` 选项在许多 PHP 版本中未对换行符做严格过滤，可构造 **CRLF 注入（`\r\n`）**：

例：![](Pasted%20image%2020260905114705.png)


**生成的实际 HTTP 请求头：**

例：
![](Pasted%20image%2020260905114723.png)
攻击者可借此伪造身份验证 Header、绕过内网 IP 白名单验证




3. ==关键区别：任意类实例化 vs 反序列化利用==

安全测试中经常出现 `SoapClient` 触发 SSRF，但两者的执行链存在本质差异：

|**维度**|**任意类实例化 (new $class)**|**反序列化利用 (unserialize)**|
|---|---|---|
|**触发阶段**|执行构造函数 `__construct()` 时立即触发|反序列化后，调用了对象的**未知方法**时通过 `__call()` 触发|
|**HTTP 请求方法**|**HTTP GET**（用于拉取 WSDL 文件）|**HTTP POST**（用于发起实际的 SOAP 请求）|
|**是否依赖魔术方法链**|不需要，只需执行 `new` 语句|必须依赖 `__call`，需要配合能够触发调用的 POP 链|
|**Payload 载荷能力**|主要是 GET 请求（结合 CRLF 可伪造部分请求头）|POST 请求体包含 SOAP XML，配合 CRLF 可攻击 Redis、FastCGI 等|

>**提示**：如果代码中在实例化之后**还会动态调用该对象的方法**（例如 `$obj->$method()`），则即使在非 WSDL 模式下，也可以将任意类实例化延伸为标准的 SOAP POST SSRF。




4. ==修复与防御策略==
---

1. **严格限制动态实例化的类（白名单机制）**： 禁止直接将不受信任的用户输入拼入 `new` 语句，设置明确的类名白名单
	
2. **禁用或卸载无需使用的 SOAP 模块**： 若业务系统不需要处理 SOAP 协议，在 `php.ini` 中注释掉 `extension=soap`，直接消除该内置类的利用条件。
    
3. **网络层 SSRF 纵深防御**：
    
    - 在网络边界通过防火墙或反向代理限制 Web 服务器出网权限。
        
    - 禁止服务器向私有网段（`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.1` 以及云元数据地址 `169.254.169.254`）发起主动 HTTP 请求





例：
![](d18b58574ac12f2f7196ef0c3f44eeb6_720.jpg)




### 使用Soap反序列化进行SSRF攻击
PHP `SoapClient` 反序列化 SSRF 是 Web 安全中一种经典的原生类利用技术。==在目标环境存在不安全的反序列化点（`unserialize()`）且没有现成的复杂应用 POP 链时，攻击者可直接利用 PHP 原生内置的 `SoapClient` 类，通过其 `__call()` 魔术方法向内网服务发起任意 **HTTP POST** 请求==



1. ==核心触发机制==


(1) 原生类的免包含优势

`SoapClient` 是 PHP 的内置类（需开启 `php_soap` 扩展）。反序列化利用时，目标服务器无需提前包含或自动加载特定业务类文件，只要反序列化数据中指定为 `SoapClient`，PHP 就能直接反序列化该对象。


(2) `__call()` 触发 HTTP POST

当对反序列化后的 `SoapClient` 对象调用一个**不存在或不可访问的方法**时，会自动触发 `SoapClient::__call()`。 在 **非 WSDL 模式** 下，`__call()` 会根据初始化配置向指定目标地址发起标准 SOAP 协议的 **HTTP POST** 请求


例：
![](Pasted%20image%2020260905115337.png)




(3) 触发途径（POP 链配合）

通常结合其他类的魔术方法作为入口：

- **`__destruct()` 入口**：若某个类在析构时调用了其成员变量的方法（如 `$this->handler->close()`），只需将 `$this->handler` 替换为伪造的 `SoapClient` 对象。
    
- **业务正常调用**：若业务逻辑反序列化后本就会对反序列化出的对象调用特定业务方法，只要该方法在 `SoapClient` 中不存在，即可直接触发




2. ==构造 Payload 的关键参数==

在非 WSDL 模式下，构造 `SoapClient` 时必须设置以下属性：

|**参数**|**字段**|**说明**|
|---|---|---|
|**WSDL**|`null`|必须为 `null`，强制开启非 WSDL 模式|
|**Options**|`location`|**SSRF 的目标地址**（可指定内网 IP、端口、路径）|
|**Options**|`uri`|命名空间标识（非 WSDL 模式下不可为空）|
|**Options**|`user_agent`|**核心攻击向量**：支持注入换行符 `\r\n`（CRLF 注入）|

例：
![](Pasted%20image%2020260905115436.png)




3. ==HTTP 报文与 CRLF 注入细节==

如果不注入 CRLF，`SoapClient` 默认发出的报文格式为 SOAP XML：

例：
![](Pasted%20image%2020260905115459.png)

**利用 CRLF 覆盖/走私关键字段：** 因为 PHP 底层未对 `SoapClient` 的 `user_agent` 参数中的 `\r\n` 进行剥离，攻击者可以在 `user_agent` 内部插入双换行符（`\r\n\r\n`）或自定义字段：

1. **注入自定义 Header**：如 `Cookie`、`Authorization`、`X-Forwarded-For`。
    
2. **修改表单解析类型**：插入 `Content-Type: application/x-www-form-urlencoded`。
    
3. **控制 POST Body**：插入自定义的 `Content-Length` 并紧跟 `\r\n\r\n` 截断原始请求，直接注入自拟的 POST 数据。目标接收端根据 HTTP 规范读取指定的 `Content-Length` 长度后停止解析，从而忽略末尾原始的 SOAP XML 数据



4. ==典型利用场景==

- **利用内网敏感 POST 接口**：内网管理后台或内部微服务常依赖 IP 白名单或内部 Cookie 进行权限校验，且大量操作（如修改密码、添加管理员、触发任务）仅接受 POST 请求。通过 `SoapClient` 伪造 Header 与 POST Body，可直接调用这些接口。
    
- **协议走私攻击 Redis（未授权）**： 将 `location` 设定为 `[http://127.0.0.1:6379/](http://127.0.0.1:6379/)`，在 `user_agent` 中通过 CRLF 注入 Redis RESP 文本协议指令。Redis 解析 HTTP 请求时会将无效指令报错忽略，遇到有效的文本指令时依然会顺序执行，从而达到写入定时任务、SSH Key 或 Webshell 的效果。
    
- **配合 PHP-FPM / FastCGI SSRF**：若结合特定环境的 FastCGI 监听端口（9000），可通过协议构造触发远程代码执行





5. ==防御与加固方案==
---

1. **杜绝不安全的反序列化**：
    
    - 优先使用 `json_encode()` / `json_decode()` 等结构化数据交换格式。
        
    - 若必须使用 `unserialize()`，应在反序列化选项中配置类白名单（PHP 7.0+）

2. **禁用或移除不需要的扩展**：

- 若业务不依赖 SOAP Web 服务，直接在 `php.ini` 中注销扩展

3. **网络层 SSRF 隔离**：
    
    - Web 应用容器应配置出网网络隔离策略，禁止直接访问 `127.0.0.1`、内网关键端口（如 6379、9000、2375）及云服务元数据 IP（`169.254.169.254`）。

4. **内部服务强身份验证**：
    
    - 内网服务（如 Redis、Memcached、内部 API）必须强制配置复杂密码认证，避免单纯依赖网络隔离或仅接受本地监听





例：
![](5f94dd25d799f4ddc5b0ef353c97a803_720.jpg) 




# 利用Gopher协议进行SSRF攻击(进阶)

## 使用Gopher协议对fastcgi实行SSRF攻击
利用 Gopher 协议针对 ==FastCGI（通常为 PHP-FPM）==实施 SSRF（服务端请求伪造）攻击，==核心机制是**通过 SSRF 漏洞利用 Gopher 协议的“任意字节流透传”特性，向内网未鉴权的 FastCGI 端口发送伪造的二进制数据包，动态修改 PHP 运行时配置，最终达成远程代码执行（RCE）**==



1. ==核心组件与漏洞机理==

为什么 FastCGI 会被攻击？

- **无身份认证**：FastCGI 协议（RFC 标准）设计用于 Web 服务器（如 Nginx/Apache）与后端解释器（如 PHP-FPM）之间的高性能本地通信，**协议本身完全没有内置身份认证机制**。只要能够建立 TCP 连接并发送合规的 FastCGI 二进制帧，PHP-FPM 就会无条件信任并执行。
    
- **默认监听模式**：很多部署环境下，PHP-FPM 默认监听在本地网络地址 `127.0.0.1:9000`。虽然外部公网无法直接访问，但如果服务器存在 SSRF，内网通信阻隔便被突破



为什么必须使用 Gopher 协议？

- **HTTP 协议的局限**：普通的 HTTP 请求行与 Header 具有严格的明文文本格式规范，并在数据头强制插入 `HTTP/1.1`、`Host` 等内容，无法与 FastCGI 所需的**二进制协议头**兼容，会导致 FastCGI 协议解析器报错断开。
    
- **Gopher 的“万能胶水”特性**：Gopher 协议格式极其自由（URL 结构为 `gopher://<host>:<port>/_<payload>`），在建立 TCP 连接后，它会将 URL 解码后的全部原始字节（包括 `\x00` 空字符、换行与任意二进制流）原封不动发送到目标端口，为模拟任意 TCP 协议提供了天然通道



2. ==FastCGI 数据帧构造原理==

攻击者需要按照 FastCGI 协议规范，依次拼接并发送三个关键的 FastCGI Record（记录块）

例：
![](Pasted%20image%2020260905122850.png)


每个 Record 都以固定的 8 字节 Header 开头：

- `version`：协议版本（固定为 `\x01`）
    
- `type`：Record 类型（`1` 为 Begin Request，`4` 为 Params，`5` 为 Stdin）
    
- `requestId`：请求 ID（如 `\x00\x01`）
    
- `contentLength`：后续数据体长度
    
- `paddingLength`：填充长度




关键的配置覆盖（FCGI_PARAMS）

在 `FCGI_PARAMS` 记录中，攻击者不仅伪造了 CGI 标准环境变量，更利用了 PHP-FPM 特有的配置覆盖能力：

|**关键环境变量**|**攻击注入值**|**作用与攻击目的**|
|---|---|---|
|`SCRIPT_FILENAME`|目标机上已存在的绝对路径（如 `/usr/share/php/PEAR.php`）|FastCGI 要求执行一个必须存在的 `.php` 文件，否则直接返回 404 并终止解析。|
|`PHP_ADMIN_VALUE` / `PHP_VALUE`|`auto_prepend_file = php://input`|动态覆盖 `php.ini` 配置，强制 PHP 在执行主文件前，先引入并执行标准输入流（STDIN）中的内容。|
|`PHP_ADMIN_VALUE` / `PHP_VALUE`|`allow_url_include = On`|确保流式协议封装包（如 `php://input`）被允许作为引入文件解析。|



代码注入（FCGI_STDIN）

攻击者将一段用于执行任意命令的 PHP 代码（例如 `<?php system($_GET['cmd']); die(); ?>`）封装进 `FCGI_STDIN` 记录。由于前置参数配置了 `auto_prepend_file = php://input`，PHP-FPM 在解析时会优先将 STDIN 中的代码当作 PHP 代码直接执行




3. ==完整攻击执行流程==


例：
![](Pasted%20image%2020260905123353.png)





4. ==攻击前提与约束限制==

- **协议库支持**：发起 SSRF 的底层客户端必须支持 `gopher://` 协议（如 cURL 默认开启，但部分高版本运行库或自定义配置会禁用该协议）。
    
- **通信信道为网络 TCP**：PHP-FPM 必须配置为监听网络端口（如 `listen = 127.0.0.1:9000`）。如果环境配置的是本地 Unix Domain Socket（如 `listen = /run/php/php-fpm.sock`），Gopher 无法跨越网络层直接发起连接。
    
- **已知合法 PHP 文件**：`SCRIPT_FILENAME` 必须指向目标磁盘上真实存在的文件路径。在通用攻击载荷中，常利用 Linux 系统预装的 PHP 模块文件（如 `/usr/share/php/PEAR.php`、`/usr/local/lib/php/PEAR.php`）或站点已知的入口文件（如 `/var/www/html/index.php`）




5. ==防御与缓解策略==

---

1. **改用 Unix Domain Socket（最有效）**
    
    修改 `php-fpm.conf` 或对应 pool 配置文件：

例：
![](Pasted%20image%2020260905123437.png)
杜绝开放 `127.0.0.1:9000` 网络端口，彻底阻断基于 TCP 的 SSRF 探测



2. **限制 SSRF 允许的协议方案**

在应用层对外部请求进行白名单校验，仅放行 `http` 和 `https` 协议，在客户端网络库中硬编码禁用 `gopher://`、`dict://`、`file://`：

例：
```
// PHP cURL 示例
curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_HTTP | CURLPROTO_HTTPS);
```


3. **配置客户端访问控制**

如果因架构需要必须监听 TCP，务必通过 `listen.allowed_clients` 严格限定仅允许信任的反向代理服务器 IP，防止来自本机的非预期会话










当存在fastcgi可访问的服务时，Gopher就可以通过构造恶意数据包来完成代码执行攻击。

使用以下脚本，我们可以生成一份fastcgi攻击流量
`https://gist.github.com/phith0n/9615e2420f31048f7e30f3937356cf75`

用法:`fmp.py -c <PHP代码> -p <fpm端口> <fpm地址> <php文件绝对路径> `






## 利用Gopher协议进行SSRF攻击工具(gopherus)



![](d54184be3fe2fc3f41f82991137b1606_720.jpg)


工具链接：`https://github.com/Esonhugh/Gopherus3`












































































































