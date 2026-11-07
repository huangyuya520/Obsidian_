
# "xxxx"APP API接口处存在SQL注入漏洞

## “智能水控”业务功能服务端存在SQL注入漏洞

`account` 参数未做任何处理直接拼进 where 字符串，双引号包裹，可闭合后注入。

==**漏洞根因**（调试页回显的 Api 同级控制器源码，get_bind 第55-56行）：==

```php
$bind = Db::table('e_bind_list')->
    where('account="'.$data['account'].'" ')->find();
```






### 信息搜集阶段

通过对APP数据包进行抓包，对`https://sqdkapp.xxx.edu.cn`URL进行目录扫描和**调试页泄露源码**

整理资产时发现该目录：

![](Pasted%20image%2020260914121824.png)




使用Ykit工具对该路径的数据包进行重放测试，发现输入学号请求能够回显学生所对应的宿舍楼层等相关信息(是学生以前所居住的寝室的相关水费信息)


![](Pasted%20image%2020260914123047.png)



### 构造Poc
尝试构造使用字符型测试语句尝试请求发现能够正常回显存在注入点

![](Pasted%20image%2020260914123641.png)



尝试使用`extractvalue`关键字构造”报错注入“poc：`241123110447" and extractvalue(1,concat(0x7e,(select database()))) ) #`，发现返回 HTTP 500 的 `ThinkPHP`框架调试页

尝试构建获取当前服务的”数据库“名：

![](Pasted%20image%2020260914144020.png)


尝试构建获取数据库中的所有”表“名：

![](Pasted%20image%2020260914151238.png)


### 尝试WAF绕过

1. **大小写**：全部 SQL 关键字用小写（`and`/`or`/`select`/`where`）。实测大写 `AND`、`OR`、`AND 1=1`、`sleep()` 直接 RST。
    
2. **限速**：连续请求会触发 RST 断连。脚本中加入失败重试 + 2~4 秒退避；手工测试每次间隔1秒以上

---


### 尝试在库获取一些重要资产

接下来尝试构造获取当前业务数据库中的所有表名：`and extractvalue(1,concat(0x7e,(select substr(group_concat(table_name),1,31) from information_schema.tables where table_schema=database()))) ) #`


![](Pasted%20image%2020260914153645.png)



进一步尝试获取 `e_config` 配置表中的字段信息：

![](Pasted%20image%2020260914162723.png)




尝试以`id`为表示读取`keyValue`的值：

![](Pasted%20image%2020260914163227.png)


通过切换`id`值遍历找到`keyName`值，进而找到水卡支付`keyName`对应的`keyValue`的值：

![](Pasted%20image%2020260914164205.png)


![](Pasted%20image%2020260914164516.png)

![](Pasted%20image%2020260914222242.png)

这是**热水充值支付回调的验签密钥**


通过以上方法获取电费支付`keyName`对应的`keyValue`的值：

![](Pasted%20image%2020260914165609.png)


![](Pasted%20image%2020260914165706.png)


![](Pasted%20image%2020260914220720.png)

`4FaSoZvtCIkVETQwvnVnGEhXxDsK8z61`




## “电表充值”业务功能服务端存在SQL注入


### 验证Poc
尝试构造使用字符型测试语句尝试请求发现能够正常回显存在注入点：

![](Pasted%20image%2020260914171027.png)



![](Pasted%20image%2020260914171702.png)




![](Pasted%20image%2020260914171913.png)


==**(“电表充值“实际服务功能所使用数据库与“水表充值”所使用的数据库相同)**==
































































































































