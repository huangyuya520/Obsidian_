
# 绕过php disable_functions机制技巧
`disable_functions` 是核心的安全配置选项，用于在 `php.ini` 中禁用特定的高危函数，防止恶意脚本或未授权用户通过执行系统命令、读写敏感文件来危害服务器安全


## php disable function机制介绍

### 补：php.ini文件简介
`php.ini` 是 PHP 运行时的核心配置文件。它==用来控制 PHP 的性能、安全、资源限制和功能开关==。PHP 在启动时会读取这个文件。你可以通过修改它来调整 PHP 的工作方式，而不需要改动代码

- 本质上就是一种配置PHP运行时行为的配置文件


主要作用

- **功能开关**：决定开启或关闭某些功能（如报错显示、文件上传）。
- **资源限制**：限制内存使用量、脚本最大执行时间和上传文件大小。
- **加载扩展**：开启需要的功能模块（如 `mysqli`、`gd` 等）


常见配置项

- `memory_limit`：每个 PHP 脚本可以使用的最大内存。
- `max_execution_time`：脚本运行的最长时间（秒）。
- `upload_max_filesize`：允许上传的文件的最大大小。
- `post_max_size`：通过 POST 提交数据的最大限制。
- `display_errors`：是否在网页上显示错误信息（生产环境建议关闭）

- ==配置项涵盖语言特性、资源分配、数据处理等等==


### 补：php.ini文件语法介绍
`php.ini` 是 PHP 的核心配置文件，其语法非常简单轻量，==基本格式为 `directive = value`（指令 = 值）==。文件由 PHP 在启动时解析，支持注释、布尔值、数字、字符串、常量及区块限定


基本语法规则

- **键值对赋值**：采用 `directive = value` 格式，等号前后可有空白字符。
- **注释符号**：以分号 `;` 开头的行或行内分号后面的内容会被当作注释忽略。
- **空白行**：所有空行都会被系统直接忽略


- php.ini文件语法
	- 通过键值对的方式配置
		- 如`name="kx"`
	- 键名大小写敏感
		- 如`NAME="kx"`和`name="kx"`是两个不同的配置
	- 使用分号做注释符
		- 如`name="kx"`
	- 使用布尔运算和位布尔运算符
		- 如`is start=0`或`is start=on`或`is start=False`或`is start=Yes`


数据类型与写法

- **布尔值（Boolean）**：
    - 开启：可用 `1`、`On`、`True` 或 `Yes`。
    - 关闭：可用 `0`、`Off`、`False` 或 `No`。

- **字符串（String）**：通常直接写文字，若包含特殊字符或空格建议使用双引号 `""` 包裹。
- **空值（Empty/None）**：等号后面不写任何内容，或直接使用关键字 `None`。
- **数字与常量**：支持整数、浮点数，以及 PHP 内置常量（如 `E_ALL`）或扩展常量。
- **容量单位简写**：内存或文件大小设置支持简写单位，如 `K`（Kilobytes）、`M`（Megabytes）、`G`（Gigabytes），例如 `memory_limit = 256M`


`php --ini`    查看本机环境中的php.ini文件位置



### 补：php.ini文件常见配置项介绍

- php.ini文件常见配置项介绍
	- ==文件上传相关==
		- `file_uploads`: 是否开启文件上传
		- `upload_tmp_dir`: 文件上传的临时文件目录
	- ==语言特性相关==
		- `short_open_tag`: 是否开启短标签
	- ==安全相关==
		- `magic_quotes_gpc`: 是否转义引号
		- `expose_php`: 是否在http响应头中显示php版本


### php disable function机制详介
PHP 的 `disable_functions` 是一种用于提升服务器安全性的配置机制，它允许管理员在 `php.ini` 中禁用特定的高危、系统级或敏感函数（如 `exec`、`system`、`passthru` 等），防止恶意脚本直接操控操作系统

==特性1：只能用于`disable`内置函数==
==特性2：必须被设置于`php.ini`==

==常见被禁用的高危函数==

- **命令执行类**：`system`、`exec`、`shell_exec`、`passthru`、`popen`、`proc_open`
- **程序进程类**：`pcntl_exec` 等进程控制函数
- **动态加载类**：`dl`（防止动态加载自定义的危险 C 扩展）



![](11d0b62ad7e8edfdf3c020e38c0d1f69_720.jpg)


## 利用第三方组件对`php disable function`机制进行绕过

### 利用`ImageMagick`组件对`disable function`机制进行绕过执行RCE==（php<=3.3）==

#### PHP ImageMagick组件库介绍
`Imagick` 是一个功能强大的 PHP 扩展，==用于调用 [ImageMagick](https://www.imagemagick.org/) 库来实现图像的创建、编辑、合成以及格式转换==。它支持超过 200 种图像格式（包括 PNG、JPEG、GIF、WEBP、PDF 和 SVG 等），能够满足 Web 应用中复杂的动态图像处理需求

主要功能与优势

- **格式支持广泛**：能够无缝读取、转换和写入数百种常见的位图与矢量格式。
- **核心图像操作**：支持对图片进行缩放（Resize）、裁剪（Crop）、旋转（Rotate）、翻转（Flip）以及锐化和模糊等滤镜处理。
- **高级合成与特效**：支持添加文字水印、多图层合并、颜色调整及复杂的图形绘制。
- **性能与面向对象**：相比传统的 GD 库，`Imagick` 基于面向对象设计（如 `Imagick` 类及其方法），在处理大图、复杂滤镜和多帧 GIF 动图时功能更为全面、稳定

- ImageMagick
	- ImageMagick是PHP中用于处理图片文件的第三方库
- 跟PHP的关系
	- php中存在一个基于ImageMagick实现的库
	- ==在低版本`（<=3.3）`下存在命令执行漏洞==


#### ImageTragick RCE（CVE-2016-3714）
ImageMagick 依赖 `delegates.xml` 配置文件处理特定格式。在老版本中，处理 HTTPS、EPHEMERAL、MSL、MVG 等格式时，命令行参数格式化字符串直接拼接了用户传入的未过滤文件名或 URL，从而导致直接注入 shell 命令


**POC（MVG / SVG 格式）例：**
![](39b04a7f4a932b1863ce5f3634235b0e_720.jpg)

**执行流程**：PHP 的 `new Imagick('exploit.mvg')` 解析到 `fill 'url(...)'` 时，ImageMagick 会调用 `curl`/`wget` 委托，拼接命令生成类似 `sh -c "wget -q -O ... '[https://example.com/image.jpg](https://example.com/image.jpg)"|id > /tmp/pwned.txt" ..."` 的指令，直接绕过 PHP 函数层执行系统命令


### 利用php `imap`组件对disable function机制进行绕过执行==RCE(php<8.4)==

#### php imap组件库简介==(php<8.4)==
IMAP 是==用于连接邮件服务器并接收、管理邮件的工具库==。==它可以操作 IMAP、POP3 和本地邮箱==。注意，==PHP 官方内置的 IMAP 扩展自 PHP 8.4 起已被弃用==，开发者现在通常转向功能更强的第三方面向对象类库（如 `webklex/php-imap` 或 `barbushin/php-imap`）

核心功能

- **连接认证**：支持通过 SSL/TLS 安全连接主流邮箱（如 QQ、Gmail、163 等）。
- **邮件检索**：支持按条件搜索、统计邮件总数与未读数。
- **内容解析**：获取邮件主题、发件人、正文及结构。
- **附件管理**：下载、保存和处理邮件携带的附件。
- **状态维护**：标记邮件为已读、移动邮件或将其彻底删除


常见应用场景

- **自动化办公**：定时抓取指定邮箱的业务回执或订单邮件。
- **客服系统**：将用户发到客服邮箱的信件自动转入工单系统。
- **邮件客户端**：搭建 Web 端的简易收信、读信网页应用

- 什么是imap
	- `imap`的全称是`InternetMailAccessProtocol(交互邮件访问协议)`
	- PHP中存在一个名为`php-imap`的扩展，实现了imap的各项功能
	- `php-imap`中某个函数会使用`rsh/ssh`来作远程`shel1`连接


![](80121fd077d2d434bba3153d29062c44.jpeg)


#### imap RCE（CVE-2018-19518）
`imap` 扩展绕过 `disable_functions` 的核心技术主要基于 ==**CVE-2018-19518**（UW c-client 远程命令执行漏洞）==。该扩展在底层封装了 University of Washington 的 `c-client` 邮件库，当通过 `imap_open()` 建立连接时，底层的参数注入缺陷允许攻击者借由系统级 `ssh` / `rsh` 客户端直接派生并执行操作系统命令，完全脱离 PHP 解释器的函数禁用规则


==核心漏洞原理（CVE-2018-19518）==

- **底层调用链路**： PHP 的 ==`imap_open()`== 会调用底层的 `mail_open()` 函数。如果服务器地址未严格过滤，且连接目标需要使用 rsh/ssh 身份验证通道，`c-client` 库会自动调用系统中的 `/usr/bin/ssh`（或 rsh）建立代理连接。

- **参数注入（Argument Injection）**： 在拼接 SSH 命令行时，`c-client` 未对传入的主机名进行严格校验。攻击者可以在邮箱服务器连接字符串中注入 `-oProxyCommand=...` 参数。

- **ProxyCommand 特性**： OpenSSH 客户端的 `-oProxyCommand` 选项用于指定连接服务器所使用的代理命令。SSH 在启动时会在底层 subshell 中直接执行该命令指定的任意系统指令


- `imap_open`函数
	- 正是`imap-php的imap_open`函数会调用`rsh`，并将用户提供的`host`直接填入



**`payload:`**
![](b8dfae57ddb53fd65cbe97b31886e426.jpg)
==由于 `c-client` 的参数解析机制会以空格作为分隔符截断命令，直接写入带空格的 Shell 命令会导致解析失败。实战中通常使用制表符（`\t`）或 **`${IFS}`** 代替空格，配合 **Base64 编码** 执行任意复合指令==




### 利用php `pcntl`组件对disable function机制进行绕过执行RCE


#### php pcntl组件库简介
**pcntl**（Process Control，进程控制）扩展==允许开发者在 PHP 中实现类似 Unix/Linux 风格的多进程创建、程序执行、信号处理和进程中断==。它常用于编写高性能的后台守护进程、命令行脚本任务分发和异步处理

核心特性与功能

- **创建子进程 (`pcntl_fork`)**：实现多进程并发处理，将一个进程复制为父进程和子进程。
- **信号处理 (`pcntl_signal` / `pcntl_async_signals`)**：捕获和处理操作系统发送的信号（如 `SIGINT`、`SIGTERM`），用于实现平滑重启或安全退出。
- ==**执行程序 (`pcntl_exec`)**：在当前进程空间执行指定的程序文件。==
- **进程等待与回收 (`pcntl_wait` / `pcntl_waitpid`)**：等待或返回子进程的终止状态，防止产生僵尸进程（Zombie Process）


使用限制与注意事项

- **仅限 CLI 与类 Unix 系统**：只能在 Linux 或 macOS 等 Unix 样式的系统下运行，**不支持 Windows**。
- **禁止用于 Web 环境**：绝不能在 Web 服务器（如 Apache/Nginx 的 PHP-FPM 模块）中启用或调用，否则会导致严重的不稳定和安全隐患

==常用核心函数==

- `pcntl_fork()`：创建子进程。
- `pcntl_waitpid(<$pid>, <$status>)`：等待特定子进程退出并回收资源。
- `pcntl_signal(<$signo>, <$handler>)`：为指定信号安装回调处理函数。
- `pcntl_alarm(<$seconds>)`：设置定时器信号

![](3bb7932e13b1626548fe12570df02afd_720.jpg)


#### pcntl RCE
`pcntl`（Process Control）扩展直接将 Unix 系统的进程控制接口（如 `fork`、`execve`、`waitpid` 等）封装为 PHP 函数。当运维人员仅在 `disable_functions` 中禁用了常规的 `system`、`exec`、`shell_exec` 等函数，却遗漏了 `pcntl` 扩展中的函数时，==攻击者可直接通过 `pcntl_exec` 在系统底层调用 `execve` 系统调用，实现不受 PHP 解释器限制的命令执行==


==核心执行函数：`pcntl_exec`==

==`pcntl_exec()` 对应 C 语言的 `execve()` 系统调用，用于在当前进程空间内加载并运行指定的二进制文件==。

- `pcntl_exec()`函数语法：
```
pcntl_exec(<$path>, <$args>, <$envs>)
```
该函数结果返回布尔值
- `$path`：必须是可执行文件的绝对路径（如 `/bin/bash`、`/bin/sh`、`/usr/bin/python` 等），不支持像 `system()` 那样直接传入裸命令字符串。
- `$args`：传递给程序的参数数组（对应 `argv`）。
- `$envs`：传递给程序的环境变量数组



- **进程替换特性**：`pcntl_exec()` 执行后会完全替换当前 PHP 进程的代码段、数据段和堆栈。如果直接在 Web 进程中调用，执行完成后该子进程会直接退出，可能导致 Web 服务器返回 HTTP 502 错误且无法直接回显命令执行结果。
- **回显获取与稳定性方案**：通常配合 `pcntl_fork()` 创建子进程执行目标命令，父进程等待执行完成并通过临时文件或网络套接字获取结果


例：
![](Pasted%20image%2020260824154427.png)



## 利用Linux `LD_PRELOAD` bypass执行RCE(`LD_PRELOAD` 环境变量劫持（配合子进程派生）)

###  补：c程序的生命周期
==(略，请有一定C语言相关知识后再观看本章)==


![](3ac6e1415f9d74aee8ec4e64f25a9d17_720.jpg)


### 补：动态链接库概念简介
动态链接库（Dynamic-Link Library，缩写为 **DLL**，在 Linux 下通常称为 **Shared Object** 即 `.so` 文件）是==一种包含可由多个程序同时使用的代码和数据的共享库文件==。==程序在运行时才载入这些库，而不是在编译时把代码全部复制到可执行文件中==

- 什么是动态链接库
	- ==是程序运行时才会被动态加载的库==
	- ==能够有效地节省空间==
- 所以...
	- 如果黑客能够控制动态链接库
	- 及其加载过程？


核心特点

- **节省空间**：多个程序调用相同时，内存中只需保留一份核心代码。
- **便于更新**：升级功能时只需替换对应的库文件，无需重新编译所有主程序。
- **独立编译**：支持模块化开发，不同功能由不同团队编写和维护

常见平台文件扩展名

- **Windows**：`.dll`
- **Linux / Unix**：`.so`
- **macOS**：`.dylib`


![](32af252f49b53053e8bcdb99fa28791b.jpg)


### 补：Linux——`LD_PRELOAD` 环境变量简介
`LD_PRELOAD` ==是 Linux 和 Unix 系统中的一个环境变量==。它能在程序运行前优先加载指定的动态链接库（`.so` 文件），==让你可以拦截、修改或替换程序中调用的系统函数和库函数，而不需要重新编译原程序的代码==

- 什么是`LD_PRELOAD`
	- 是Linux中的一个环境变量，用于影响程序运行时加载哪些库
	- ==会在程序运行前被优先加载==
	- 因此`LD_PRELOAD`可以被用于`Hook Linux系统`命令
		- `Hook`：指通过某种手段==劫持==某些方法/函数的执行流
		- 添加自定义执行逻辑


主要特点与工作原理

- **优先加载**：动态链接器在启动程序时，会最先读取 `LD_PRELOAD` 中指定的库。
- **函数劫持**：如果自定义的库里包含了与原程序相同名称的函数，系统会优先使用自定义库里的函数。
- **无需源码**：它允许你在不修改、不重新编译软件源码的情况下，直接改变程序的行为


常见应用场景

- **软件调试与测试**：用来追踪、拦截特定的 API 调用，检查内存泄漏或分析性能瓶颈。
- **功能修补**：在不更新主程序的前提下，临时修复某些有问题的系统函数（类似热补丁）。
- **安全研究与注入**：安全测试中用于模拟或拦截敏感函数；但部分恶意软件也会利用它来隐藏行踪或进行权限提升

### 补：Linux——readelf命令简介
`readelf` 是 Linux 系统中==用于显示 ELF（可执行与链接格式）文件详细信息的命令行工具==。它支持 32 位和 64 位二进制文件，常用于软件开发、逆向工程和底层调试

- 什么是readelf
	- 是Linux中用来显示一个或者多个e1f格式的目标文件的信息的工具
- 什么是elf文件
	- ELF 文件（**可执行与可链接格式**，英文全称 **Executable and Linkable Format**）是 ==Linux 和类 Unix 系统上通用的**标准二进制文件格式**==。它用来存储程序的目标代码、可执行程序、共享库以及核心转储文件，类似于 Windows 系统下的 `PE` 格式（如 `.exe` 和 `.dll` 文件）
	- 内部包含一个可执行文件的各种信息




- 语法：
	- **查看文件头**：  
	    `readelf -h <文件名>`

	- **查看节区信息**：  
	    `readelf -S <文件名>`

	- **查看依赖的动态库**：  
	    `readelf -d <文件名>`


用例：
![](0d89f87db36414bde33a589238373d7b_720.jpg)



### 思路：利用Linux——`LD_PRELOAD`环境变量劫持某些Linux自带方法

- 思路的前提
	- 1、==PHP是基于C实现，本身会调用大量的内置库函数==
	- 2、LD_PRELOAD这样的环境变量可以被PHP代码设置
		- ==如`putenv("LD PRELOAD=<xxxx>")`==
- ==利用的前提==
	- ==需要存在一个可以将我们的恶意动态链接库放置到目标服务器上==


### `LD_PRELOAD` RCE
==利用 Linux 的 `LD_PRELOAD` 环境变量劫持动态链接库==，Linux 加载动态链接库时，若设置了环境变量 `LD_PRELOAD`，系统会在加载其他库之前优先加载该变量指定的 `.so` 文件。当该进程启动时，动态链接器会自动执行共享库中的`__attribute__((constructor))` 初始化函数

**触发机制**：

1. PHP 通过 `file_put_contents()` 在可写目录（如 `/tmp`）写入一个恶意的 `.so` 动态链接库。
2. ==调用 `putenv("LD_PRELOAD=/tmp/evil.so")` 设置环境变量。==
3. 使用 `Imagick` 处理需要调用外部解析器（Delegate）的文件类型（例如 PDF、PS、EPS、SVG、WMF、MPG 等）。
4. 底层 `libmagick` 会调用 `fork()` / `execve()` 启动外部命令（如 Ghostscript 或 ffmpeg），子进程在初始化阶段加载恶意 `.so` 并执行其中的系统命令

例：
![](Pasted%20image%2020260824165519.png)


#### 构造`LD_PRELOAD` 环境变量劫持C语言代码


##### 利用 `__attribute__((constructor))` 构造函数
无需关心目标子进程具体调用了哪些 `libc` 函数，只要子进程初始化加载该库，即自动执行构造函数


例：
![](Pasted%20image%2020260824165211.png)


##### 劫持通用 C 库函数（如 `getuid` / `geteuid`）
重写子进程必经调用的函数，在函数被触发时注入恶意逻辑

例：
![](Pasted%20image%2020260824165352.png)





## 基于PHP自身的缺陷绕过php disable_functions机制技巧


### 补：php垃圾回收机制简介
PHP 的垃圾回收机制==主要由**引用计数（Reference Counting）** 和 **循环引用回收器（Concurrent Cycle Collector）** 组成==。当变量的引用计数降为 0 时内存会被立即释放；当遇到复杂的对象或数组循环引用时，则通过将可能根放入缓冲区并在满时触发同步算法来清理内存

- 什么是垃圾回收机制
	- 垃圾回收机制（Garbage Collection）用于自动管理内存对象，清除无用对象
	- 关键参数介绍
		- 引用计数：一个PHP变量被引用的次数，存储在PHP引擎中的zval变量中

核心组成部分

- **引用计数**：每个 PHP 变量存储在 `zval` 容器中，包含一个引用计数器。当变量被赋值、传递或引用时计数增加，取消引用或离开作用域时计数减少。计数变为 0 时内存立即释放。
- **循环引用**：当数组或对象内部引用了自身或彼此，导致它们的引用计数减不到 0，常规引用计数无法释放它们。
- **根缓冲区（Root Buffer）**：当引用计数减少但未到 0 的复合类型变量会被视为“可能根”，放入缓冲区。
- **同步回收算法**：当缓冲区满时，PHP 运行深度优先搜索，对查找到的循环引用变量做计数减操作，标记并释放无用的垃圾内存

![](a4bc63a90fa68a68acf6572c9af746c1.jpeg)



### 补：UAF漏洞简介(PWN相关领域漏洞)
**UAF（Use-After-Free，释放后使用）** 是一种经典的==堆内存破坏漏洞（CWE-416）==。其根本原因在于程序释放（`free` 或 `delete`）了某块堆内存后，**未将指向该内存的原指针置为 `NULL`**（形成**悬空指针 / Dangling Pointer**），并在后续流程中再次通过该悬空指针引用已释放的内存

- UAF漏洞简介
	- UAF (Use After Free)
- 漏洞根因：
	- 释放了一个堆块后没有将相应的指针置为NULL，产生悬空指针
	- 攻击者如果通过某种方式则有可能控制该指针
	- 这里的某种方式可以是malloc相同大小的内存

**核心触发流程**

==UAF 的产生与利用通常依赖现代内存分配器（如 glibc `ptmalloc`）优先复用大小相近内存块（LIFO 机制）的特性==

![](Pasted%20image%2020260824201925.png)

1. **分配（Allocate）：** 程序申请内存块 `Chunk 1`，指针 `ptr_A` 指向该结构体（内含数据或函数指针）。
2. **释放（Free）：** 程序释放了 `Chunk 1`，但 `ptr_A` 依然保留原地址，成为悬空指针。
3. **重分配/占坑（Reallocate）：** 攻击者构造大小与 `Chunk 1` 匹配的恶意对象 `Chunk 2`。内存分配器将原 `Chunk 1` 的内存重新分给 `ptr_B`，攻击者借此控制该内存区域的内容。
4. **使用（Use）：** 程序再次调用 `ptr_A`（例如 `ptr_A->callback()`），此时读取到的其实是攻击者写入的恶意数据


示例：
![613](b4742e8f93a887e47d813f42f3c4e208_720.jpg)



### 利用php GC缺陷导致的漏洞对php disable_functions机制进行绕过技巧==(PHP<7.4+)==

利用 PHP 垃圾回收（Garbage Collection, GC）缺陷绕过该机制的核心思路是：==**通过触发 GC 逻辑缺陷造成 UAF（Use-After-Free），构建任意内存读写原语，进而泄漏 `libc` 基址并直接调用底层 `system` 执行系统命令**==

- PHP GC UAF漏洞描述
	- https://bugs.php.net/bug.php?id=72530
- PoC:
	- https://github.com/mmOr1/exploits/blob/master/php7-gc-bypass/exploit.php

==**核心漏洞原理**==

PHP 使用**引用计数（Refcount）+ 循环垃圾收集器**管理内存。当数组或对象之间产生循环引用时，引用计数无法归零，Zend 引擎会将这些可能存在垃圾的变量加入 `GC Root Buffer`。

当调用 `gc_collect_cycles()` 或缓冲区满时，`GC` 会遍历节点并尝试减少计数。如果引擎在遍历、清理或对象析构过程中存在逻辑缺陷（例如析构函数在清理过程中被执行并重新引用了内存，或复合类型引用更新不同步），就会导致：

- 某块内存已被释放（Free）。
    
- PHP 代码中仍然持有指向该内存地址的悬垂指针（Dangling Pointer）

==**典型利用链流程**==

1. **堆排布与触发 UAF (Heap Feng Shui & Trigger)** ==构造深度嵌套的循环引用结构==（如包含自身引用的数组或对象），填满 GC 缓冲区后触发 `gc_collect_cycles()`，使目标 `zval` 结构被释放，但外部变量依然指向该地址。
2. **内存占坑与伪造数据 (Heap Grooming / Fake Zval)** ==通过分配特定长度的字符串或数组申请内存，精准复用刚刚被释放的内存块==。通过控制写入的内容，直接覆写该内存块的元数据，伪造一个非法的 `zval` 结构（例如伪造为 `IS_STRING` 并修改其 `len` 字段为一个极大值）。
3. **构建任意内存读写与信息泄露 (Information Leak)** ==利用篡改了超长长度的伪造字符串，实现越界读取==（Out-of-Bounds Read）：
    
    - 读取 PHP 进程内存，获取 Zend 内部数据结构和函数指针。
        
    - 解析 ELF 结构或定位已加载模块的 GOT 表，计算出 `libc` 的基地址，最终解析出 `libc.so` 中的 `system` 函数真实内存地址。
4. **控制流劫持执行命令 (Control Flow Hijacking)**
    
    - 构造一个伪造的闭包对象（`Closure`）或自定义对象，将其底层的 `handler` / 函数执行指针直接指向 `libc` 的 `system`。
        
    - 触发该伪造对象调用，传入要执行的 Shell 命令（如 `/bin/sh -c ...`），从而脱离 PHP 层的函数禁用检查

![](99b30fd0ca05f39bf4df7118d908a8e2.jpg)


### 利用php Backtrace缺陷导致的漏洞对php disable_functions机制进行绕过技巧==（PHP 7.0 – 7.4 部分小版本）==
利用 PHP **Backtrace（调用栈回溯）缺陷**绕过 `disable_functions` 的本质，同样是==利用底层的内存破坏漏洞（UAF，Use-After-Free）构造任意内存读写原语，通过篡改内存中的函数指针直接调用 `libc` 的 `system` 函数，从而绕过应用层的函数黑名单==

- PHP Backtrace UAF漏洞描述
	- https://bugs.php.net/bug.php?id=76047

- PoC:
	- https://github.com/mmOr1/exploits/blob/master/php7-backtrace-bypass/exploit.php

==**核心漏洞原理**==

在 PHP 中，`debug_backtrace()` 或 `Exception::getTrace()` 用于生成当前执行栈的回溯信息，包括函数名、调用文件及**传入的参数列表（`args`）**。

- **引用计数与参数捕获不同步**：当 `debug_backtrace()` 构建参数数组时，引擎需要将当前执行帧（`zend_execute_data`）中的参数封装为 `zval` 存入回溯数组。
    
- **析构与悬垂指针**：==在特定 PHP 7.x 版本中==，若参数包含特定复合结构（如引用、特定对象或触发类型转换/析构回调），在回溯数组构建或后续操作中，目标变量的内存被提前释放（Free），但回溯数组或局部符号表中仍然保留着指向该内存地址的指针，从而产生 **UAF 漏洞**

**典型利用链流程**

1. **构造特定调用栈与捕获参数** 定义多层嵌套函数调用，在参数中传递精心构造的变量（如包含引用的数组），并在内层函数中调用 `debug_backtrace()` 提取调用栈帧中的 `args`。
2. **触发 UAF 释放目标内存** 通过在特定时机 `unset` 原始变量或操作回溯数组，使得底层对应的 `zval` 内存被释放并归还给 Zend 内存管理器（ZendMM），但在回溯数组或变量作用域中该指针未被置空。
3. **堆喷射与伪造 Zval (Fake Zval)**
    
    - 利用字符串分配（`str_repeat` 等）精准申请与刚刚释放相同大小的内存块，将可控的二进制数据写入被释放的内存区域。
        
    - 将该内存伪造成一个合法的 `zval` 结构体（例如类型设置为 `IS_STRING`，长度字段 `len` 设为极大值 `0x7fffffff`，指针指向任意内存地址）。
4. **信息泄露与地址定位 (Information Leak)**
    
    - 利用超长伪造字符串实现**任意地址读取**。
        
    - 读取 PHP 进程内存空间，遍历 ELF Header 定位 PHP 或已加载模块的 GOT 表，获取 `libc.so` 的加载基址。
        
    - 从 `libc` 导出符号表中解析出 `system` 函数的实际内存地址。
5. **劫持执行流执行命令**
    
    - 构造一个伪造的 `Zend Closure`（闭包对象），将其底层执行函数指针（`invoke` / `handler`）覆盖为 `system` 的地址。
        
    - 执行该伪造闭包，并传入要执行的 Shell 指令（如 `/bin/sh -c "id"`），直接在 OS 层创建子进程执行命令



# 绕过php open_basedir机制手法技巧
==`open_basedir` 是 PHP 的一项核心安全机制==。它将 PHP 脚本可读写的文件的活动范围限制在指定的目录树中，有效防止恶意脚本读取或修改系统敏感文件（如 `/etc/passwd`）以及跨站读取其他网站的数据。当启用该功能时，所有涉及文件操作的函数都会进行路径检查


## 补：php open_basedir机制简介
==`open_basedir` 是 PHP 的一项核心安全机制==。==它将 PHP 脚本可读写的文件的活动范围限制在指定的目录树中，有效防止恶意脚本读取或修改系统敏感文==件（如 `/etc/passwd`）以及跨站读取其他网站的数据。当启用该功能时，所有涉及文件操作的函数都会进行路径检查


==工作原理==

- **路径检查**：当调用 `include`、`fopen`、`file_get_contents` 等文件函数时，PHP 会检查目标文件路径是否在 `open_basedir` 指定的列表内。
- **匹配规则**：多个目录之间用冒号（Linux）或分号（Windows）隔开。如果目标路径不在授权范围内，操作会被拒绝并报错。
- **生效范围**：不仅限制文件本身，还包括符号链接等指向的目标


==常见配置方式==

- **全局配置**：在 `php.ini` 中设置 `open_basedir = /www/wwwroot/site1/:/tmp/`。
- **虚拟主机配置**：在 Nginx/Apache 的配置文件或每个站点的独立 PHP 配置中动态指定。
- **目录级配置**：通过网站根目录下的 `.user.ini` 文件进行限制


==优缺点与注意事项==

- **安全性高**：在多用户共享服务器（如虚拟主机、面板环境）中，能有效隔离不同用户的网站目录。
- **性能影响**：每次文件系统调用都会增加额外的路径解析和字符串对比开销，高并发下会带来轻微的性能损耗。
- **兼容性问题**：部分开源框架或程序需要读取临时目录、缓存目录或系统全局组件，配置过严会导致报错或白屏


- open_basedir介绍
	- open_basedir用于限制php脚本可以访问的目录


![](f37fd41ab44d0fb26349a961112f0396.jpg)



## 补：php glob协议简介
==PHP 的 `glob://` 是一种**数据流包装器（Stream Wrapper）**，自 PHP 5.3.0 起引入==。它==允许通过类似 `glob()` 的通配符模式来查找匹配的文件路径==，主要配合诸如 `iterator_apply`、`include` 或其他支持数据流的函数使用

==核心特性与限制==

- **不支持读写**：`glob://` 协议仅用于匹配路径，**不支持**读取、写入或追加文件内容。
- **不受配置限制**：它不受 `allow_url_fopen` 和 `allow_url_include` 的影响。
- **无状态支持**：不支持 `stat()` 等文件属性查看操作

- 语法：
```
$<it> = new DirectoryIterator("glob://<ext/spl/examples/*.php>");
```

==协议限制与安全属性==

- **只读/受限**：`glob://` 仅用于查找匹配的文件路径，不支持写入、修改、删除或读取文件内容。
- **不受配置限制**：它不受 PHP 配置文件中 `allow_url_fopen` 和 `allow_url_include` 的限制，因为它是针对本地文件路径模式的查找

- glob协议介绍
	- glob是一种文本匹配模式
	- 在php中可以作为数据流包装器，==php5.3+开始生效==
	- 能够用于批量地匹配文件（本质就是利用通配符这个范式）


例：
![](4a14f626d698009d1c10ab0a4a24b088.jpg)


## 补：php DirectoryIterator类简介
PHP 的 [DirectoryIterator](https://www.php.net/manual/zh/class.directoryiterator.php) 是标准 PHP 库 (SPL) 中==用于面向对象遍历文件系统目录的内置类==。它继承自 `SplFileInfo` 并实现了迭代器接口，能高效、清晰地获取目录中的文件与子文件夹信息

==主要特点==

- **面向对象**：相比传统的 `opendir` 和 `readdir` 函数，代码结构更清晰。
- **结合文件信息**：每个遍历出来的项本身就是一个 `SplFileInfo` 对象，可直接调用文件大小、修改时间等方法。
- **内存友好**：按需读取目录项，不会一次性把所有文件名加载进庞大的数组中

==常用方法==

- `__construct($path)`：传入目标目录路径，初始化迭代器。
- `getFilename()`：获取当前项的文件名。
- `isDot()`：检查当前项是否为 `.` 或 `..` 目录，遍历时通常需要排除。
- `isDir()` / `isFile()`：判断是目录还是文件。
- `getSize()` / `getMTime()`：获取文件大小和最后修改时间

- 语法：
```
$<dir> = new DirectoryIterator('</path/to/dir>');
```


- Directorylterator介绍
	- 是PHP中用于查看文件系统目录内容的类
	- https://www.php.net/manual/en/class.directoryiterator.php

例：
![](b770892dc9dbf078801d2fd624fa60eb.jpg)


## 补：符号链接与symlink函数介绍

- 什么是符号链接
	- Symboliclink,也叫软链接
	- 是一种特殊的文件，包含一条通过绝对路径或者相对路径指向别的文件的引用
	- `Linux`语法：
	`ln -s <源文件> <链接文件>`
- symlink函数
	- php中用于创建符号链接的函数


### 符号链接机制简介（软链接）
符号链接（Symbolic Link，又称==软链接==）是==一种特殊的文件。它包含一段文本，记录着另一个文件或文件夹的路径。当系统打开这个链接时，会自动跳到目标位置==。==它和 Windows 的快捷方式很像==

==工作原理==

- **独立文件**：符号链接有自己的独立存储空间和文件属性。
- **路径指向**：它只保存目标文件的绝对路径或相对路径。
- **透明访问**：程序读取链接时，系统会重定向到真正的数据

==核心特点==

- **跨分区支持**：可以在不同的磁盘或分区之间创建。
- **支持目录**：能为文件夹（目录）创建链接。
- **依赖目标**：如果删除了源文件，链接就会失效（变成“死链接”）。
- **不增引用数**：它不会增加源文件的引用计数

==与硬链接的区别==

- **硬链接**：指向同一个文件的存储区块（inode），源文件删除后数据仍在。
- **符号链接**：只指向路径，源文件删除后链接损坏


### symlink函数简介
==PHP 的 `symlink()` 函数用来为目标文件或目录创建一个符号链接（快捷方式），成功时返回 `true`，失败时返回 `false`==

- 语法：
```
symlink(<$target>, <$link>);
```
- **`target`**：符号连接指向的目标文件或目录的路径。
- **`link`**：要创建的符号连接文件的路径和名称

==使用注意==

- **返回值**：操作成功返回 `true`，失败返回 `false`。
- **平台差异**：在较旧的 PHP 版本或特定 Windows 环境下支持有限（现代 Windows 和主流 Linux/Unix 系统均支持）。
- **权限安全**：服务器可能出于安全考虑在 `php.ini` 中禁用此函数（通过 `disabled_functions` 配置），或因目录没有写权限导致创建失败

![](37ed6b3c90c09bd0e1a60c68c3f12843.jpeg)


## 补：php SpIFileInfo类简介
`SplFileInfo` 是一个内置的面向对象类，==用于获取和操作单个文件或目录的详细信息（如大小、修改时间、扩展名等）==。它可以代替传统的 `pathinfo`、`filemtime` 等面向过程函数，让文件处理更规范

==核心功能与特点==

- **面向对象**：用对象的方法代替分散的文件函数。
- **路径灵活**：初始化时传入路径即可，文件甚至可以暂时不存在。
- **配合 SPL 迭代器**：常与 `DirectoryIterator` 结合使用，高效遍历目录

- 语法：
```
$<info> = new SplFileInfo('</path/to/file.txt>');
```

==常用方法==

- `getFilename()`：获取文件名。
- `getExtension()`：获取文件后缀名。
- `getSize()`：获取文件大小（字节）。
- `getMTime()`：获取最后修改时间。
- `isDir()`：判断是否为目录。
- `isFile()`：判断是否为文件。
- `openFile()`：将当前文件转换为 `SplFileObject` 对象以读写内容


- SplFilelnfo类简介
	- 是php中用于文件操作的类，提供了一系列的高级接口
	-  其中的==`getRealPath`==具备一个有趣的特性


![](490d040e8ee38236181d57f156c85681.jpg)



## 补：php ini_set函数简介
[ini_set](https://www.php.net/manual/zh/function.ini-set.php) 是一个内置函数，==用于在脚本运行时临时修改 `php.ini` 中的配置选项。修改在函数执行后立即生效，并在当前脚本运行结束时自动恢复默认值==

- ini_set函数简介
	- ini_set函数用于设置php.ini中的某些配置项

==核心特点==

- **作用域**：仅在当前脚本生命周期内有效，不影响其他正在运行的脚本或全局配置。
- **语法格式**：`ini_set(string $option, string|int|bool $value)`
- **返回值**：成功时返回旧的配置值，失败或不支持修改时返回 `false` 或空。
- **限制条件**：并非所有的配置选项都能通过该函数修改。配置的可修改范围（如 `PHP_INI_ALL` 或 `PHP_INI_USER`）决定了它是否可用

- 语法：
```
ini_set(<$option>, <$value>)
```
- `$option`：要设置的配置选项名称（例如 `display_errors`、`max_execution_time`）。
- `$value`：选项的新值

==常见应用场景==

- **开启错误显示**：在调试代码时临时打开页面错误提示。
- **调整内存限制**：处理大文件或复杂计算时临时调大内存。
- **修改执行超时**：延长脚本的最大运行时间


![](e6f09d8625d7538b5af7c66443f6422a.jpg)


## 利用glob协议配合文件操作类/函数列目录绕过php openbasedir机制技巧


### glob协议+DirectoryIterator类方法 RCE
利用 `glob://` 伪协议配合 `DirectoryIterator` 类是绕过 PHP `open_basedir` 限制==进行**目录遍历与敏感文件嗅探**==；攻击者通常将其作为侦察手段，通过定位临时文件、Session 文件或敏感配置，与文件包含（LFI）等漏洞结合形成利用链以实现 RCE

**核心绕过原理**
- **检查机制脱节**：`glob://` 封装协议底层依赖系统的 `glob()` 实现或 Zend 内部模式匹配逻辑。在部分 PHP 版本与特定 SPL 迭代器组合中，`glob` 在检索并返回文件列表时未对每个匹配项逐一校验 `open_basedir`。
	
- **SPL 类配合**：PHP 的 `DirectoryIterator`、`FilesystemIterator` 或 `GlobIterator` 允许传入 URI 模式。当传入 `glob:///path/*` 时，SPL 迭代器能够遍历出受限目录外的所有文件名和目录名

例：
![](ce1c57e9b2abc6ce243127dfd9cdfb02_720.jpg)


### glob协议+opendir和readdir函数 RCE
==`glob://` 伪协议配合 `opendir()` 与 `readdir()` 函数，利用了 PHP 底层流包装器（Stream Wrapper）在处理通配符匹配时缺失 `open_basedir` 路径完整校验的缺陷==。该方法能够跨目录列举受限区域的文件名与目录名，通过精确捕获目标路径（如 Session、临时上传文件、日志等）与文件包含（LFI）漏洞组合，最终实现 RCE

==**核心绕过原理**==

PHP 的 `open_basedir` 通常在文件系统 I/O 函数（如 `fopen`、`file_get_contents`、`scandir`）执行时检查绝对路径前缀。

- **Stream Wrapper 检查缺陷**：当以 `glob://` 协议传递路径给 `opendir()` 时，PHP 会调用 `glob` 流包装器的目录打开句柄（`dir_opendir`）。
    
- **元数据列举逃逸**：底层的 `glob()` 处理通配符模式并生成匹配项链表，在通过 `readdir()` 逐个读取文件名时，未对每个匹配项逐一执行 `open_basedir` 白名单校验，从而绕过了目录枚举的限制



例：
![](88a67f89e96b26ab238fa2715c9085d4_720.jpg)


### glob协议+scandir函数 RCE
==`scandir()` 结合 `glob://` 协议同样是利用 PHP 内置的 `glob` 流包装器（Stream Wrapper）在处理通配符匹配时缺失 `open_basedir` 逐项校验的缺陷==。==与 `opendir()` 逐项读取不同，`scandir()` 会一次性将匹配模式的文件名读取并以数组形式返回==，大幅简化了受限目录的枚举过程，为后续定位关键文件并打通 RCE 利用链提供支撑

==**核心绕过原理**==

`scandir()` 在底层的实现中封装了 `php_stream_opendir`、`php_stream_readdir` 以及排序逻辑。

- **包装器分发**：当传入 `glob:///path/*` 时，Zend 引擎将路径解析权移交给 `glob` 流包装器。
- **边界校验缺失**：底层 `glob` 处理模式匹配时，仅根据模式规则检索文件系统的元数据并构建结果列表，未对匹配出的每个文件绝对路径进行 `open_basedir` 白名单检查。
- **直接返回数组**：`scandir()` 接收到句柄后直接读取所有条目，并以 PHP 数组形式返回匹配到的所有文件和文件夹名称


例：
![](Pasted%20image%2020260824225834.png)


## 利用php SpIFileInfo类中的`getRealPath`方法列目录绕过php openbasedir机制技巧
利用 `SplFileInfo::getRealPath()` 绕过 PHP `open_basedir` 是==一种**基于布尔状态反馈（Boolean-based Oracle）的路径与文件盲测探测技巧**==
与 `glob://` 直接返回匹配列表不同，`getRealPath()` 本身不具备模式匹配遍历功能，但它在解析绝对路径时不严格受 `open_basedir` 拦截；==攻击者通过字典爆破或逐字符探测确认受限目录下的文件与目录存在性，随后配合本地文件包含（LFI）等漏洞打通 RCE 利用链==

==**核心漏洞原理**==

==`open_basedir` 机制通常在脚本尝试打开、读取或修改文件时触发检查。==

- **底层实现脱节**：`SplFileInfo` 是 SPL 标准库中用于获取文件元数据的类。其 `getRealPath()` 方法底层调用 C 运行时的 `realpath()` 解析规范化的绝对路径。
	
- **布尔状态差异**：在部分 PHP 版本中，`getRealPath()` 在检测底层 inode 并解析路径时，并未在第一时间拦截受限路径并抛出 Warning：
    
    - **目标文件/目录存在**：函数成功解析并返回该文件的绝对路径字符串。
        
    - **目标文件/目录不存在**：函数返回 `false`。
        
- **盲测原语形成**：攻击者利用这一返回值差异，构建自动化脚本对受限文件系统进行字典穷举或字符爆破，从而完全刺探出受限目录内的文件树

poc：
![](1f4bc76fa2f6896c1fa54e42425a0665_720.jpg)



## 利用php symlink函数配合chdir和mkdir方法列目录绕过php openbasedir机制技巧
利用 `symlink()` 配合 `chdir()` 和 `mkdir()` 是 PHP 中绕过 `open_basedir` 限制的经典漏洞利用手法。不同于仅能探测文件名的 `glob://` 协议，该方法利用了 PHP 虚拟当前工作目录（Virtual Current Working Directory, VCWD）在处理相对路径与符号链接时的==**检查与解析时序差异（TOCTOU 特性）**，能够**直接读取受限目录外的任意文件内容、列举目录，并可直接配合 `include` 触发本地文件包含（LFI）达成 RCE**==

==**核心绕过原理**==

PHP 的 `open_basedir` 机制通过 VCWD 对路径进行规范化校验，确保目标路径落在允许的前缀范围内。

1. **创建相对软链接**：攻击者在当前合法目录下创建指向自身的软链接 `tmplink`。
    
2. **构建跨目录软链接**：调用 `symlink("tmplink/../../../../etc/passwd", "target")`。在创建 `target` 链接时，PHP 对目标路径进行 `open_basedir` 校验：由于 `tmplink` 此时指向当前合法目录，路径前缀符合白名单规则，**成功通过安全检查并创建软链接**。
    
3. **消除中间层并实体化**：删除软链接 `tmplink`（`unlink("tmplink")`），并新建同名的实体文件夹（`mkdir("tmplink")`）。
    
4. **路径重定向逃逸**：当后续读取 `target` 时，系统沿文件系统路径解析：`./tmplink/../../../../etc/passwd` 直接回溯到了根目录下的 `/etc/passwd`，而 PHP 的底层文件访问接口由于之前已通过符号链接合法性校验，从而直接跨出 `open_basedir` 范围

poc：
![](676c8dfb968fb5c4bf1f64bdf8aba982_720.jpg)


## 利用php chdir函数与ini_set函数修改open_basedir机制绕过php openbasedir机制技巧
利用 `chdir()` 与 `ini_set()` 动态修改 `open_basedir` 是 PHP 中经典的==**配置收缩逻辑漏洞**==。该方法==利用了 PHP 在处理相对路径 `..` 时对目录层级边界校验的设计缺陷，通过构造临时目录逐级向上回溯，最终将 `open_basedir` 的限制彻底重置为系统根目录 `/`，使攻击者获得全局文件读写能力并直接串联 LFI 实现 RCE==

==**核心漏洞原理**==

PHP 允许在脚本运行时通过 `ini_set('open_basedir', ...)` 修改限制，但有一个安全前提：**新设定的路径必须在原 `open_basedir` 允许的范围内（只能收紧，不能扩大）**。

1. **相对路径合法性校验误判**： 当在当前合法目录下新建子目录 `sub` 并 `chdir('sub')` 后，调用 `ini_set('open_basedir', '..')`。PHP 检查 `..`（即上一级目录，原合法目录）确实落在当前的 `open_basedir` 许可范围内，因此**允许此次修改**，将 `open_basedir` 的值直接更新为字符串字面量 `..`。
    
2. **虚拟工作目录（VCWD）逃逸**： 当 `open_basedir` 被设置为相对路径 `..` 后，PHP 内部的 Virtual Current Working Directory 模块在每次执行 `chdir('..')` 时，都会依据“当前目录的上一层”进行合法性判断。
    
3. **循环回溯至根目录**： 连续调用多次 `chdir('..')`，工作目录会沿着物理文件树一路向上回溯直至系统根目录 `/`。
    
4. **彻底解封**： 当当前工作目录已经到达系统根目录 `/` 时，再次调用 `ini_set('open_basedir', '/')`，此时由于当前目录就是 `/`，该设置被判定为合法，从而将整个操作系统的根目录设为白名单，彻底废除 `open_basedir` 限制


poc：![](3862b5fbbf18824de3ddc0dcfef2c89b.jpg)







































































































































































































































































































































































