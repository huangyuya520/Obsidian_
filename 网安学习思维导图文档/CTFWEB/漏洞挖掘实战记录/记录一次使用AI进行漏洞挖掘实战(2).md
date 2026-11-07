
# "xxxx"APP API接口处存在支付链路伪造漏洞


## “智能水控”业务功能服务端存在支付链路伪造漏洞

### 漏洞原理

![](Pasted%20image%2020260914173232.png)


### 信息搜集阶段

在对“智能水控”业务功能服务端使用SQL注入漏洞获取数据库中的电费支付`keyName`对应的`keyValue`的值后点击进入软件的”智能水控“页面，选择进入”一、二期水控“


![238](Pasted%20image%2020260914173931.png)


创建充值订单，随便定义一个充值金额：

![244](Pasted%20image%2020260914174040.png)



从yakit抓包工具查看响应包一侧一些数据：

创建订单反馈数据包信息，发现响应数据的`payurl`链接：
![](Pasted%20image%2020260915133546.png)


访问`payurl`链接发现订单号等信息：

![](Pasted%20image%2020260915134032.png)














使用“智能水控”业务功能服务端存在的SQL注入漏洞获取`e_logs`表中的日志字段与`e_config` 配置表中水卡支付`keyName`对应的`keyValue`的值(热水充值支付回调的验签密钥)，发现寻找到data json格式的数据日志：


![](Pasted%20image%2020260914221629.png)


![](Pasted%20image%2020260914222148.png)


![](Pasted%20image%2020260914221819.png)


字段说明：

- ==`orderno`：Step 1 的订单号（**带W前缀**），这是服务端定位订单的唯一依据；==
    
- `orderstate`：字符串 `"1"` 表示支付成功；
    
- ==`amount`：与订单金额一致（数值型）；==
    
- `jylsh`（交易流水）/ `tradeno`（平台交易号）：自造唯一值即可，服务端未校验其真实性；
    
- `payerid`/`payername`：服务端不校验与订单账号的对应关系（实测沿用日志中他人值也能通过）；
    
- ==`xmpch`/`paymethod`/`return_type`：照抄真实回调值。==





因为在创建订单付款时会触发服务器中的“计算签名”功能以验证订单真实性，所以需要明白签名校验算法逻辑，签验校验算法经过大量计算得出规律：==`md5(参数ksort拼接 + e_config.PayWaterKey)`==：




首先尝试在`make_order`订单创建请求包中进行测试，将账户传入的字符以数组的形式传入`account[]=xxx`，发现响应包中存在thinkphp调试页面泄露(`app_debug=true`)：
![](Pasted%20image%2020260915112610.png)



make_sign 源码：ksort → `k=v&` 拼接 → `key=appid`，且错误上下文变量里直接带出了 `WateAppid = 2FB69B2F...`(HTML title 属性里是完整值）
![](Pasted%20image%2020260915122101.png)


![](Pasted%20image%2020260915122309.png)




其次尝试在`make_order`订单创建请求包中进行测试，不传入`account`的参数，发现响应包中也存在thinkphp调试页面泄露：


`make_order` 开头 + `getAccountInfo/preOrder` 调用结构；
`make_order` 中段：金额校验、`preOrder` 调用、`trade_no` 生成 `'W'.date('ymdHis').rand(1111,9999)`，该部分为订单号生成原理结构代码
![](Pasted%20image%2020260915122546.png)





其次尝试将`callback`订单反馈请求包变形为GET请求方法，不传入任何参数，发现响应包中也存在thinkphp调试页面泄露：


发现签名算法逻辑源码：
![](Pasted%20image%2020260915123312.png)


**==综上所诉可得：==**
验签算法（ApiWater.php 第216-218行 + Pay::getSignContent）：

```
signStr = ksort({"op","xmpch","version","data","requesttime"}) 后按 "k=v" 用 & 连接
sign    = md5(signStr + PayWaterKey)     → 转大写
```

即参与签名的参数为 **data、op、requesttime、version、xmpch 五个**（按字典序 data 在前），**不含 sign 自身**；data 的值是**未 URL 编码的原始 base64（含 padding）**。

```python
params = {'op':'Notify', 'xmpch':'143-2025090004', 'version':'1.4', 'data':b64, 'requesttime':ts}
s = '&'.join(f'{k}={params[k]}' for k in sorted(params))
sign = hashlib.md5((s + KEY).encode()).hexdigest().upper()
```

可以尝试使用服务端数据库中的`e_log`表单中的日志中真实回调信息验证计算
如：
（id=255007，订单 W2609141001509999）
复算应得到 `87BAF1BBBD5FFFD220840D64A2D758B1`

与日志 `sign`变量对应的参数 一致即算法正确









### Poc构造验证


尝试构造支付请求数据包

访问`payurl`链接获取订单号：

![](Pasted%20image%2020260915134108.png)



对构造回调 data json进行相应数据构造并对其进行base64编码：


先获取`e_logs`数据表中的指定人员的`orderdate`字段的表单创建时间值和`orderno`字段的表单值，以及使用**密钥 + 相应签名算法**进行计算`SIGN`签名值
![](Pasted%20image%2020260915140024.png)




构造后再以post方式进行发送：

![](Pasted%20image%2020260915150718.png)


在响应成功后，再去水控页面进行查看是否充值成功：

充值前：
![](Pasted%20image%2020260915151032.png)


充值后：
![](Pasted%20image%2020260915151059.png)



发现余额变化，证明可形成完整支付链条
![296](Pasted%20image%2020260915151005.png)





## “电表充值”业务功能服务端存在支付链路伪造漏洞

**==(漏洞原理与“智能水控”业务功能服务端存在支付链路伪造漏洞逻辑相同)==**


### 信息收集

创建电费支付订单抓包时发现相应的请求路径与`payurl`：

![](Pasted%20image%2020260915200508.png)





与上述“智能水控”业务漏洞一样，访问url，拿订单号，`e_logs`数据表中的指定人员的`orderdate`字段的表单创建时间值和`orderno`字段的表单值，以及使用**密钥 + 相应签名算法**进行计算`SIGN`签名值





### 构造Poc



构造POST请求包信息：

![](Pasted%20image%2020260915200852.png)


发送后查看APP中的钱包金额是否发生变化，变化则充值成功


![519](Pasted%20image%2020260915201009.png)























































