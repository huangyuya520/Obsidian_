

# Linux应急排查
在实战护网（攻防演练）的蓝队防守中，Linux 主机应急排查的核心目标是**快速定位失陷主机、止血遏制扩散、提取攻击载荷、还原攻击路径**。

面对安全设备（EDR、态势感知、WAF）告警或发现异常行为时，可遵循以下标准化应急排查流程。


==应急处置原则：先遏制，后分析==
遇到失陷主机时，**切忌直接重启系统或直接 `kill -9` 进程**。直接重启会导致内存易失数据丢失、抹除内存马，甚至触发红队预埋的启动后门；直接杀死进程可能触发守护进程自拉起或告警红队。

- **网络层遏制**：在核心交换机或防火墙上下发安全策略阻断该主机对内网其他资产的访问，或仅阻断恶意 C2 IP 的出向流量。
    
- **进程挂起（保活取证）**：如果需要阻止恶意进程继续活动，使用 `kill -STOP <PID>` 挂起进程（暂停执行），保留 `/proc/<PID>` 下的完整内存映射与文件句柄，便于后续逆向分析。



==一、网络连接与异常外联排查==

红队攻入 Linux 主机后，通常会建立反弹 Shell、C2 远控信道或拉取后门工具，排查网络往往是定位问题最快的切入点。


1. 检查异常活动连接与监听端口

例：
```
# 查看所有建立的 TCP 连接及关联进程（推荐优先使用 ss）
ss -antulp | grep ESTAB

# 检查当前监听的高危或未知端口
ss -lntup
```


2. 重点排查特征
- **可疑外联 IP**：重点关注连接到境外、非业务公网 IP、动态域名或云厂商高位端口的 `ESTABLISHED` 连接。
    
- **常见反弹 Shell 进程名**：注意 `sh`、`bash`、`zsh`、`python`、`perl`、`nc`、`socat` 等原生解释器直接与外部公网建立的 TCP 连接。

- **通过端口定位可疑进程**：

例：
```
lsof -i :<恶意连接端口>
lsof -p <可疑进程PID>
```




==二、异常进程深度排查==
1. 资源占用排查（挖矿木马与僵尸网络）
例：
```
# 查看 CPU 占用前 10 的进程
ps aux --sort=-%cpu | head -n 11

# 查看内存占用前 10 的进程
ps aux --sort=-%mem | head -n 11
```

2. 检查被删除的恶意二进制文件（Memory-only 执行）
红队常在落地执行木马后立即 `rm` 掉源文件，实现“无文件”驻留。系统仍在运行但磁盘文件已标记为 `(deleted)`：

例：
```
ls -al /proc/*/exe 2>/dev/null | grep '(deleted)'
```


**提取内存中的样本**：若发现恶意程序已被删除，可直接从 `/proc` 文件系统将正在运行的二进制文件 dump 出来用于分析：

例：
```
cp /proc/<PID>/exe /tmp/recovered_malware.bin
```



3. ==分析进程上下文==

例：
```
# 查看进程执行的完整命令行（防止进程名伪装成 [kworker] 等内核线程）
cat /proc/<PID>/cmdline | tr '\0' ' '

# 查看进程运行时的环境变量（常包含攻击者使用的临时变量、代理配置）
cat /proc/<PID>/environ | tr '\0' '\n'

# 查看进程打开的文件描述符（套接字、日志文件、临时文件）
ls -l /proc/<PID>/fd
```



==三、后门账号与特权提升排查==
攻击者攻破系统后，常创建影子账户或滥用 `sudo` 提权以维持访问权限。


1. 检查超级权限账号（UID 为 0）

例：
```
# 查看系统中所有 UID=0 的特权用户（除 root 外通常不应存在）
awk -F: '$3==0{print $1}' /etc/passwd
```


2. 检查具有登录权限的非常规用户

例：
```
# 过滤掉无法交互式登录的用户
grep -v -E "nologin|false" /etc/passwd
```


3. 排查特权与 Sudo 配置

例：
```
# 查看当前 sudoers 配置及附加包含目录
cat /etc/sudoers
ls -la /etc/sudoers.d/
```
重点排查是否存在带有 `NOPASSWD: ALL` 的可疑条目。


4. 排查 SSH 公钥后门与登录会话

例：
```
# 检查 root 及常规用户的公钥认证文件
cat /root/.ssh/authorized_keys
cat /home/*/.ssh/authorized_keys

# 查看当前在线用户与历史登录记录
w
last -n 20       # 成功登录历史
lastb -n 20      # 暴力破解与失败登录历史
```


==四、持久化与自启动机制排查==
Linux 下的驻留机制非常多样，排查时需系统性扫查以下载体：

1. 定时任务（Crontab）

定时任务是红队最常用的权限维持手段，需排查用户级与系统级两类：

例：
```
# 检查当前用户及指定用户 crontab
crontab -l

# 排查系统底层与目录型计划任务
cat /etc/crontab
ls -la /etc/cron.*
ls -la /var/spool/cron/crontabs/
```


2. 系统服务与 Systemd 单元

例：
```
# 检查最近修改或新创建的 systemd 服务
ls -lat /etc/systemd/system/ | head -n 15
ls -lat /lib/systemd/system/ | head -n 15

# 查看处于运行状态的未知第三方服务
systemctl list-units --type=service --state=running
```


3. 环境与 Shell 初始化脚本

攻击者常在登录触发脚本中注入反弹 Shell 或载荷拉取代码：

- 用户级：`~/.bashrc`、`~/.bash_profile`、`~/.profile`、`~/.bash_logout`
    
- 全局级：`/etc/profile`、`/etc/profile.d/`、`/etc/bash.bashrc`、`/etc/environment`



4. 传统自启动脚本

例：
```
# 检查 rc.local
cat /etc/rc.local 2>/dev/null
ls -la /etc/init.d/
```



==五、文件系统与落地物排查==

1. 重点临时与共享目录

恶意脚本与木马通常释放到具有全局可写执行权限的目录中：

例：
```
ls -lat /tmp /var/tmp /dev/shm
```
_注：`/dev/shm` 是纯内存文件系统，很多无盘驻留木马首选释放在该路径。_



2. 时间范围追溯（根据告警时间点反查）

若告警发生于过去 2 小时内，搜索该时间窗口内新增或被修改的文件：

例：
```
# 查找过去 120 分钟内修改过的常规文件（排除系统动态目录）
find / -mmin -120 -type f 2>/dev/null | grep -v -E "/proc/|/sys/|/run/"
```


3. SUID/SGID 特权后门

检查是否有攻击者赋予 SUID 权限以留作提权后门的二进制工具：

例：
```
find / -perm -4000 -type f 2>/dev/null
```


4. Web 资产 WebShell 与篡改排查

若受害机器为 Web 服务器：

- 检查网站根目录下最近修改的脚本：

例：
```
find /var/www/html/ -name "*.php" -mtime -3
find /var/www/html/ -name "*.jsp" -mtime -3
```
- 借助工具（如河马、D 盾等 WebShell 查杀引擎）或使用代码版本控制（`git status`/`git diff`）对比代码基线。


==六、系统日志审计与证据链还原==

1. 系统认证与安全日志

- **RHEL/CentOS**：`/var/log/secure`
    
- **Debian/Ubuntu**：`/var/log/auth.log`

例：
```
# 提取成功的 SSH 登录 IP 与用户
grep "Accepted password" /var/log/auth.log | awk '{print $1,$2,$3,$9,$11}'
grep "Accepted publickey" /var/log/auth.log | awk '{print $1,$2,$3,$9,$11}'

# 提取 sudo 执行提权命令的日志
grep "sudo:" /var/log/auth.log
```


2. 命令历史记录（Bash History）

例：
```
# 查看 root 历史命令
cat /root/.bash_history

# 检查当前会话历史配置是否被篡改（如 HISTSIZE=0 或 HISTFILE=/dev/null）
echo $HISTFILE
echo $HISTSIZE
```


==七、进阶排查：动态链接库劫持与 Rootkit==
当常规命令看不到异常，但网络流量持续告警时，需怀疑存在反取证隐蔽技术。

1. 动态链接库劫持（LD_PRELOAD）

攻击者通过将恶意 `.so` 写入全局预加载文件，Hook 常用命令（如 `ps`、`netstat`、`ls`）以隐藏其进程与网络连接：

例：
```
# 检查预加载配置文件是否被污染
cat /etc/ld.so.preload 2>/dev/null

# 检查环境变量中是否存在劫持
env | grep -i preload
```



2. 命令完整性校验

如果怀疑 `ps`、`netstat`、`lsof`、`top` 已被红队替换篡改：

例：
```
# 检查核心二进制文件的包完整性（以 RedHat/CentOS 为例）
rpm -Vf /bin/ps /bin/netstat /usr/bin/top

# Debian/Ubuntu 系列
debsums -c 2>/dev/null
```



3. 可疑内核模块（LKM Rootkit）

例：
```
# 检查当前加载的内核模块
lsmod

# 重点排查无模块依赖、命名可疑的外部内核模块
```



==应急闭环：报告与复盘关键要素==

完成现场取证与止血后，护网防守方需输出溯源报告，重点闭环以下要素：

1. **失陷原因（入口点）**：是弱口令、0day/Nday 漏洞利用、还是口令复用？
    
2. **影响范围（横向移动）**：攻击者是否以此主机为跳板扫描了内网其他网段？
    
3. **恶意资产清单**：整理所有攻击载荷的 MD5/SHA256、外部 C2 IP、域名与端口，同步下发防火墙/EDR 全网封禁并全盘核查。



## 系统信息收集

- 查看版本内核信息：`cat /etc/os-release | /proc/version | unname -a`

- 账号安全 
	- 1、用户信息文件目录 `/etc/passwd` 
		- `root:x:0:0:root:/root:/bin/bash`
		- `account:password:UID:GID:GECOS:directory:shell`

		- 7 个字段详细介绍
			- **用户名 (Username)**
			    - 登录系统时使用的名字。
			    - 区分大小写，具有唯一性。 [[1](https://www.ibm.com/docs/zh/aix/7.3.0?topic=passwords-using-etcpasswd-file)]

			- **密码占位符 (Password)**
			    - 早期 Linux 系统在此处存储加密后的密码。
			    - 出于安全考虑，现代 Linux 系统将真正的加密密码移到了 `/etc/shadow` 文件中。这里通常显示一个字母 `x`（表示密码存放在 shadow 中），有时也可能是 `*` 或 `!`（表示账号被锁死或禁用了密码登录）。 [[1](https://www.ibm.com/docs/zh/aix/7.3.0?topic=passwords-using-etcpasswd-file), [2](https://blog.csdn.net/a1154490629/article/details/52190801)]

			- **用户标识号 (UID - User ID)**
			    - 系统的数字身份标识。
			    - `0`：系统超级管理员 `root` 的 UID。
			    - `1 ~ 999`（不同发行版略有差异）：系统保留的系统用户/服务账号（如 `bin`、`daemon`、`nobody` 等）。
			    - `1000及以上`：普通用户账户。 [[1](https://www.ibm.com/docs/zh/aix/7.3.0?topic=passwords-using-etcpasswd-file), [2](https://blog.csdn.net/a1154490629/article/details/52190801)]

			- **组标识号 (GID - Group ID)**
			    - 用户所属的默认用户组（Primary Group）的数字 ID，对应 `/etc/group` 文件中的记录。 [[1](https://www.ibm.com/docs/zh/aix/7.3.0?topic=passwords-using-etcpasswd-file)]

			- **注释性描述 (GECOS / Comment)**
			    - 记录用户的真实姓名、联系电话、办公室地址等个人附加信息，通常由 `finger` 等工具使用。 [[1](https://www.ibm.com/docs/zh/aix/7.3.0?topic=passwords-using-etcpasswd-file), [2](https://blog.csdn.net/a1154490629/article/details/52190801)]

			- **用户主目录 (Home Directory)**
			    - 用户登录系统后默认进入的家目录路径（例如 `/root` 或 `/home/username`）。 [[1](https://www.ibm.com/docs/zh/aix/7.3.0?topic=passwords-using-etcpasswd-file), [2](https://blog.csdn.net/a1154490629/article/details/52190801)]

			- **登录 Shell (Login Shell)**
			    - 用户登录后系统为其启动的 Shell 程序路径（常见的如 `/bin/bash` 或 `/bin/zsh`）。
			    - 如果该字段被设置为 `/sbin/nologin` 或 `/bin/false`，则代表该系统用户**不允许**登录交互式 Shell（常用于保障系统服务安全）。
		-    ==用户名：密码：  用户ID:组ID:用户说明:家目录:登陆之后的 shell==
		- ==注意：无密码只允许本机登陆，远程不允许登陆==

	- 2、用户口令文件 /etc/shadow
		- `root:$6$oGs1PqhL2p3ZetrE$X7o7bzoouHQVSEmSgsYN5UD4.kMHx6qgbTq wNVC5oOAouXvcjQSt.Ft7ql1WpkopY0UV9ajBwUt1DpYxTCVvI/:16809:0:99999:7:::`
		- 用户名：加密密码：密码最后一次修改日期：两次密码的修改时间间隔： 密码有效期：密码修改到期到的警告天数：密码过期之后的宽限天数：账号失效时间：保留字段

		- 9个字段详解
			- 文件每行通过冒号（`:`）分隔为 9 个字段： [[1](https://blog.csdn.net/happyjacob/article/details/109701906)]

			- **登录名（Username）**：用户的账户名称，与 `/etc/passwd` 一致。 [[1](https://xixiliguo.github.io/linux/linux-shadow.html), [2](https://www.cnblogs.com/klb561/p/10574726.html)]

			- **加密密码（Encrypted Password）**：经单向加密算法处理后的密码字符串。如果以 `$6$` 开头表示 SHA-512 算法；若为空、`!` 或 `*` 开头，则表示没有密码或账户被锁定无法直接登录。 [[1](https://xixiliguo.github.io/linux/linux-shadow.html), [2](https://blog.csdn.net/happyjacob/article/details/109701906)]

			- **最后修改时间（Last Modified Date）**：自 1970 年 1 月 1 日算起到密码最后一次被修改的天数。 [[1](https://xixiliguo.github.io/linux/linux-shadow.html), [2](https://blog.csdn.net/happyjacob/article/details/109701906)]

			- **最小修改间隔（Minimum Interval）**：密码修改后必须经过的最小天数，在此天数内用户不能再次修改密码。 [[1](https://xixiliguo.github.io/linux/linux-shadow.html), [2](https://blog.csdn.net/happyjacob/article/details/109701906)]

			- **最大有效期（Maximum Validity Period）**：密码保持有效的最大天数，超过此天数则必须强制更换密码。 [[1](https://xixiliguo.github.io/linux/linux-shadow.html), [2](https://blog.csdn.net/happyjacob/article/details/109701906)]

			- **警告天数（Warning Days）**：密码过期前提前多少天向用户发出警告提示。 [[1](https://xixiliguo.github.io/linux/linux-shadow.html), [2](https://blog.csdn.net/happyjacob/article/details/109701906)]

			- **非活动期（Grace Period / Inactive）**：密码过期后，账户允许处于非活动状态并仍可登录去修改密码的天数，超时则彻底禁用账户。 [[1](https://xixiliguo.github.io/linux/linux-shadow.html), [2](https://blog.csdn.net/happyjacob/article/details/109701906)]

			- **账号失效日期（Account Expiration Date）**：自 1970 年 1 月 1 日算起到账户被强制停用的绝对天数。 [[1](https://xixiliguo.github.io/linux/linux-shadow.html), [2](https://blog.csdn.net/happyjacob/article/details/109701906)]

			- **保留字段（Reserved）**：系统保留未用，供未来扩展。



- 系统信息收集
	- `who`   查看当前登录用户（tty 本地登陆 pts 远程登录） 
	- `w`   查看系统信息，想知道某一时刻用户的行为 
	- `uptime`   查看登陆多久、多少用户，负载状态


- 重点关注以下目录：
	- `/tmp/`
	- `/home/`
	- `/var/tmp`
	- `/root/`



## 入侵排查思路

- 1、查询特权用户特权用户(uid 为0) 
	- `[root@localhost ~]# awk -F: '$3==0{print $1}' /etc/passwd` 

- 2、查询可以远程登录的帐号信息 
	- `[root@localhost ~]# awk '/\$1|\$6/{print $1}' /etc/shadow`

- 3、除root帐号外，其他帐号是否存在sudo权限。如非管理需要，普通帐号应删 除sudo权限 
	- `[root@localhost ~]# more /etc/sudoers | grep -v "^#\|^$" | grep "ALL=(ALL)"` 

- 4、禁用或删除多余及可疑的帐号 
	- `usermod -L <user>`   禁用user帐号，帐号无法登录，`/etc/shadow` 第二栏为 ! 开头 
	- `userdel <user>`   删除 user 用户 
	- `userdel -r <user>`   将删除 user 用户，并且将 `/home` 目录下的 user 目录一并删除




## 历史命令

- 通过 `.bash_history` 文件查看帐号执行过的系统命令 
- root 用户的历史命令：`history` 
- 入侵排查： 
	- 进入用户目录下，导出历史命令。 
	- `cat .bash_history >> history.txt`




## 检查异常端口

- 使用 netstat 网络连接命令，分析可疑端口、IP、PID 
- `netstat -antlp | more` 
- 查看下 pid 所对应的进程文件路径，
	- 运行 `ls -l /proc/$PID/exe` 或 `file /proc/$PID/exe`（$PID 为对应的 pid 号）





## 检查异常进程
- 使用 ps 命令，分析进程 
- `ps aux | grep pid` 
- 系统运行级别示意图： 
	- 查看运行级别命令：`runlevel`



![](Pasted%20image%2020261004142612.png)



## 检查开机启动项


- 系统默认允许级别 
	- `vi /etc/inittab` 
	- `id=3：initdefault` #系统开机后直接进入哪个运行级别 
- 开机启动配置文件 
	- `/etc/rc.local` 
	- `/etc/rc.d/rc[0~6].d` 
- 当我们需要开机启动自己的脚本时，只需要将可执行脚本丢在 `/etc/init.d` 目录下，然后在`/etc/rc.d/rc*.d` 文件中建立软链接即可。


注：
![](Pasted%20image%2020261004143844.png)



## 启动项文件
- 常见命令: 
	- `more /etc/rc.local` 
	- `/etc/rc.d/rc[0~6].d` 
	- `ls -l /etc/rc.d/rc3.d/`






## 检查定时任务
- 利用 crontab 创建计划任务 
- `crontab -l`   列出某个用户cron服务的详细内容 
- ==Tips：默认编写的crontab文件会保存在 (`/var/spool/cron/用户名` 例如: `/var/spool/cron/root`==
- `crontab -r`   删除每个用户cront任务(谨慎：删除所有的计划任务) 
- `crontab -e`   使用编辑器编辑当前的crontab文件

例：
![](Pasted%20image%2020261004161608.png)


- 利用 anacron 命令实现异步定时任务调度


使用案例：
![](Pasted%20image%2020261004161750.png)



## 重点关注以下目录中是否存在恶意脚本

- `/var/spool/cron/*`    存放**每个用户自己的定时任务文件**
- `/etc/crontab`     **系统的主 crontab 文件**
- `/etc/cron.d/*`    存放**系统级定时任务配置片段**，格式与 `/etc/crontab` 相同（带用户字段）
- `/etc/cron.daily/*`     每天执行一次
- `/etc/cron.hourly/*`    每小时执行一次
- `/etc/cron.monthly/*`   每月执行一次
- `/etc/cron.weekly/`     每周执行一次
- `/etc/anacrontab`    anacron 的配置文件，格式为 `周期 延迟 任务标识符 命令`
- `/var/spool/anacron/*`     存放各任务**上次执行的时间戳**（如 `cron.daily`），anacron 用它判断是否需要补跑


==可以使用`more <文件目录>`命令查看目录下所有文件==





## 检查服务自启动
- 第一种修改方法： 
	- `chkconfig [--level 运行级别] [独立服务名] [on|off]` 
	- `chkconfig –level 2345 httpd on`    开启自启动 
	- `chkconfig httpd on`    （默认level是2345） 
- 第二种修改方法： 
	- 修改 `/etc/re.d/rc.local`文件
	- 加入 `/etc/init.d/httpd start`
- 第三种修改方法： 
	- 使用 `ntsysv` 命令管理自启动，可以管理独立服务和 xinetd 服务。

- 查询已安装的服务：RPM 包安装的服务 
	- `chkconfig --list`   查看服务自启动状态，可以看到所有的RPM包安装的服务 
	- `ps aux | grep crond` 查看当前服务 
- 系统在3与5级别下的启动项 
- 中文环境 
	- `chkconfig --list | grep "3:启用\|5:启用"` 
- 英文环境 
- `chkconfig --list | grep "3:on\|5:on"` 
- 源码包安装的服务 
- 查看服务安装位置 ，一般是在`/user/local/` 
- `service httpd start` 
	- 搜索`/etc/rc.d/init.d/` 查看是否存在



## 检查异常文件
1. 查看敏感目录，如`/tmp`目录下的文件，同时注意隐藏文件夹，以`..`为 名的文件夹具有隐藏属
2. 得到发现WEBSHELL、远控木马的创建时间，如何找出同一时间范围内 创建的文件？ 
	- 可以使用find命令来查找，如 `find /opt -iname "*" -atime 1 -type f` 找出 `/opt` 下一天前访问过的文件
3. 针对可疑文件可以使用 `stat` 进行创建修改时间。





## 检查系统日志

- 日志默认存放位置：`/var/log/` 
- 查看日志配置情况：`more /etc/rsyslog.conf`


![](Pasted%20image%2020261004180211.png)


![](Pasted%20image%2020261004180231.png)


比较重要的几个日志：登录失败记录：`/var/log/btmp //lastb` 最后一次登录：`/var/log/lastlog //lastlog` 登录成功记录: `/var/log/wtmp //last` 登录日志记录： `/var/log/secure` 
目前登录用户信息：`/var/run/utmp //w、who、users` 
历史命令记录：`history` 仅清理当前用户：`history -c`






## 日志分析技巧
- 常用的shell命令 
- Linux下常用的shell命令如：`find、grep 、awk、sed`





### 补：find 命令

- 查找当前目录下所有 .txt 文件   `find . -name "*.txt"`
- 查找大于 1MB 的文件   `find /path/to/dir -size +1M`
- 查找并删除 7 天前的日志文件   `find /var/log -name "*.log" -mtime +7 -exec rm {} \;`


![](Pasted%20image%2020261004182212.png)



### 补：grep 命令
- 在文件中搜索包含 "error" 的行   `grep "error" file.txt`
- 递归搜索目录中所有 .log 文件中的 "warning"   `grep -r "warning" /var/log/*.log` 
- 显示不包含 "success" 的行   `grep -v "success" file.txt`



![](Pasted%20image%2020261004182155.png)


![](Pasted%20image%2020261004182544.png)





### 补：awk 命令

- 输出文件的第一列和第三列   `awk '{print $1, $3}' file.txt`
- 以冒号为分隔符，输出 /etc/passwd 的第一列   `awk -F: '{print $1}' /etc/passwd`
- 计算文件的行数   `awk 'END {print NR}' file.txt`


![](Pasted%20image%2020261004183014.png)


![](Pasted%20image%2020261004183022.png)



### 补：sed 命令

- 替换文件中的 "old" 为 "new"   `sed 's/old/new/' file.txt` 
- 删除包含 "error" 的行   `sed '/error/d' file.txt` 
- 直接修改文件内容   `sed -i 's/old/new/' file.txt`


![](Pasted%20image%2020261004183131.png)


![](Pasted%20image%2020261004183140.png)



### 其他在实战中常用命令

例：
![](Pasted%20image%2020261004183254.png)


#### find ./ -type f -name "*.php" | xargs grep "eval("


![](Pasted%20image%2020261004183655.png)











