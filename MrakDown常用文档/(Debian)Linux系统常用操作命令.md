### 其他常见使用命令

**sudo 使用特权模式执行命令  相当于root权限是执行的**
**sudo passwd root 设置root用户密码**
**ctrl + s  保存  ctrl+x 退出**
**apt update 更新源**
**apt install `<softwarename>`  下载资源包**
==**apt search `<softwarename>` 搜索软件**==
==**apt remove `<softwarename>` 卸载软件(标准卸载（只删除程序包，保留配置文件）)**==
**==sudo apt purge <软件名>    完全卸载（删除程序包以及相关的配置文件）==**
**==sudo apt autoremove   清理无用依赖（卸载不再需要的孤立依赖包）==**

**ctrl+alt+t 打开终端**
**cp `<filename1>` `<filename2>`  将一份文件中的内容复制一份到另一个文件中**

> [! 注意]
> （补充：**/dev/stdout**标准输出连接符号，是一个伪文件；常配合通过管道（`|`）或重定向
> 号（`>`）使用，将文件结果输出在终端中）

**mv `<oldfilename>` `<newfilename>`/`</foldername>`  将文件改名为新的文件或者将文件移动到另一个文件夹中**

**|    管道命令，适用于将多个命令连接起来的符号，每个 | 就是一个管道连接点，左侧命令写数据进去，右侧命令读出来。数据单向流动（从左到右），grep经常通过管道符”|”组合使用**
==**grep -i `<filename>`  搜索相匹配的文件并打印输出出来**==
**df -h 显示磁盘空间使用量情况**
**ls -l <文件名> 查看文件权限信息**
**python -m venv `<vnevname>`       设置python虚拟环境**
**source <venv/bin/activate>       启动python虚拟环境**
**deactivate            退出python虚拟环境**
**tar -czf <压缩文件名> <原文件名> 将一个文件打包为一个压缩文件**
**tar -xzf <压缩文件名> 将文件解压**





### 包下载构建安装与删除
==**sudo dpkg -i /.../`<filename>`.deb    安装 .deb 文件**==
**sudo apt remove `<package-name>`   卸载软件包**
**sudo apt install -f                            修复环境依赖**
**sudo apt autoremove    移除不再需要的旧依赖**
**sudo apt autoclean        清理旧的软件包缓存**




### cd 命令 

**cd ~   用户的主目录**
**cd -   进入到上一次目录 desktop**
**cd ..   返回上级目录**


### ls命令  查看当前目录的所有文件 

**ls .<文件>/<隐藏文件>/<或隐藏文件>**
**ls -al 列出所有文件，包括隐藏文件**


### 创建文件

**touch <文件名> 创建文件**
**mkdir `<foldername>` 创建文件夹**
**echo  > `<filename>` 写入内容进文件中**
**echo >> `<filename>` <追加内容> 追加写入内容进文件中**
**mkdir -p <文件夹>/<文件夹> 在文件夹中创建多个嵌套文件夹**
==**`><文件名>`        极简创建文件**： 无需 `touch`，单独使用 `>foo` 即可创建一个名为 `foo` 的空文件（仅占 4 字节）==

### 删除文件

**rm `<filename>` 删除文件**
**rm -rf `<foldername>` 强制删除目录 文件夹里面有文件也可以删除**




### 查看命令

**cat `<filename>` 读取文件内容并输出到标准输出**
**less `<filename>`   分页显示文件内容**
**tail -n `<number>`/-f `<filename>`    功能是查看文件尾部内容，例如默认会在终端界面上显示指定文件的末尾10行，如果指定了多个文件，则会在显示的每个文件内容前面加上文件名来加以区分| 
-f  持续显示文件尾部最新内容
-n  设置显示文件尾部的行数**

**head -n `<number>` `<filname>`  功能是显示文件开头的内容，默认为前10行**
**-n  设置显示文件尾部的行数**

**tac -b <分隔标志>/-s <字符串分隔标志> `<filename>`     用于反向显示文件内容，即常见的查看文件内容命令cat的反写形式**
**-b 在行前添加分隔标志**
**-s  使用字符串作为分隔标志**

**more `<filename>`    功能是分页显示文本文件的内容**

**nl -b a `<filename>`     功能是用于显示文件内容及行号**
**-b a 显示指定文件内容及行号信息，空行也加上行号**

**sort -r/-h `<filename>` 功能是对文件内容进行排序(默认降序排序)**
**-r 以相反的顺序排序(升序)**
**-h 以更易读的格式输出信息**

**uniq -c/-u `<filename>`  去除文件中的重复内容行**
**-c 显示每行在文本中重复出现的次数**
**-u 仅显示没有重复的纪录**


**more `<filename>`**
**退出按键盘q键**,软连接 相当于windows快捷方式
**pwd 查看当前路径**
**id 查看当前用户**
**arch 显示机器的处理架构**
**uname -a 显示全部信息，包括内核名等信息****
**date 显示系统日期**
**reboot 重启系统**
**hostname 查看主机名**
**df -h 查看磁盘空间**
**free -h 查看内存空间**
**cat /proc/version 显示内核版本信息**
**cat /etc/hosts 查看hosts文件**
**echo $PATH 查看当前变量**
**htop  查看系统运行的进程**
**fastfetch -c all   显示获取系统详细信息**
**lsb_release -a   显示当前Linux发行版的名称、版本号、操作系统版本以及其他相关信息**
**cat /etc/os-release  查看系统版本信息**
**service sshd status 查看SSH服务状态**
**sudo service ssh restart 重启 SSH 服务**

==`ls -ld .`     仅查看当前目录本身的详细权限信息==



### 软连接

**ln -s <源文件或目录> <目标文件或目录> 在当前路径创建一个名为 xxx 的软链接，指向 /xxx/xxx/xxx 文件夹或文件；或在 /xxx 目录下创建一个名为 xxx 的软链接，指向 /xxx/xxx/xxx 文件夹，创建软连接**
**rm -rf <软链接名称>  删除软连接**



### 查找文件

**find <位置> -name `<filename>`      从指定位置开始查找相关文件位置**
**su `<username>`**       切换用户
**whereis `<filename>`     全局查找文件及其同名帮助手册路径**
**updatedb     更新创建文件索引**
**locate `<filename>`      查找指定文件位置**
**\*      称为通配符 搜索有xxx开头的文件 .xxx 结束的文件**


### 用户相关命令
**adduser `<username>` 创建用户**
**su `<username>`    切换到`<username>`用户，root权限下不需要密码 直接可以切换**
**su root 切换root用户 需要密码**
**passwd `<username>`   修改指定用户密码**
**userdel `<username>`   删除用户账户**
**cat /etc/passwd              查看用户列表**
**cat /etc/group                查看用户组列表**
**hostnamectl     查看当前主机名**
**sudo hostnamectl set-hostname `<新主机名>`     修改为新的主机名**

 




### 网络相关命令
**netsart -tuln 查看端口信息，-t表示tcp连接；-u表示udp连接；-l表示监听中的端口；-n表示直接使用数字形式显示地址和端口号**
**ifconfig 查看网卡信息**
**ping -c 4 URL 指定ping命令回显的次数**
**curl URL    向指定 URL 发送 GET 请求，并将响应内容输出到终端**
**wget URL  下载指定网络路径的文件**
**ip addr show 显示所有网络接口的ip地址**
**ip a  显示所有网络接口的ip地址**
**whois URL   用于查找并显示有关域名、IP 地址和用户的信息**
**dig URL  查询域名解析记录与dns服务器**
**sudo systemctl restart network.service 重启网络服务**
**sudo systemctl restart NetworkManager   重启 NetworkManager 服务**


### 进程相关命令
**ps -ef 以完整的格式显示所有进程**
**ps -e --forest 以树的结构显示进程关系**
kill -9 `<PID>` 强制关闭进程
top 可动态显示 CPU、内存、进程 等资源使用情况

### 文件权限相关命令
**chmod `<number>` ./.../`<filename>` 改变文件权限设置**
**sudo chown -R \$USER:\$USER .   把当前目录 . 下所有文件的所有权递归地改回给当前登录用户**

### 用户组权限命令
**chown -R `<oldowner>`:`<nwowner>` ./.../`<filename>`  改变文件拥有者**

### 防火墙命令
**ufw status 查看防火墙状态**
**ufw allow <端口号> 通过指定端口号来允许特定的服务**
**ufw status verbose 查看防火墙所有已配置的规则**
**ufw enable 启动防火墙**
**ufw disable 禁用 UFW**
**iptables -L   查看系统防火墙过滤规则**
iptables -F   清除防火墙相关过滤规则 



### 系统服务命令
**service `<xxx>` start    启动指定的服务程序**
**service `<xxx>` stop    关闭指定的服务程序**
**service `<xxx>` status      查看指定服务的状态**
**service `<xxx>` restart     重启指定的服务程序**
**service --status-all   查看系统中所有服务的状态**
**service networl restart   重启网卡设置**
**ifconfig `<eth0>` down  关闭指定的网络设备**
**ifconfig `<eth0>` up  启动指定的网络设备**
==**systemctl start `<xxx>`   启动服务**==
==**systemctl stop `<xxx>`   停止服务**==
==**systemctl enable `<xxx>` 设置开机自启动**==

### 编辑器vi / vim命令
**键h 光标左移，键j 光标下移，键k 光标上移，键l 光标右移**
**:set nu    显示行号**
**:set nonu    取消行号**
**/<关键字>    在文件中搜索关键字，按键n显示搜索下一个**
**dd     删除行**
**按键x      删除一个字符**
**按键y      复制**
**yy            复制一行**
**按键p      粘贴**
**按键u      撤销上一个操作** 
**ctrl+r       取消上一个撤销操作**

### 系统磁盘命令
**fdisk -l 查看磁盘命令**


### 补：==Linux中常见的分隔符==
- Linux中具备多种分隔符
	- 分号（`;`）：多条命令的分隔符，命令会连续执行下去
	- 与（`&&`）：多条命令的分隔符，连续执行直至命令失败终止
	- 或（`||`）：多条命令的分隔符，连续执行直至命令成功终止
	- 单个与（`&`）：效果同分号
	- 管道符号（`|`）：前一条命令的执行结果输出给后一条命令
- 其他特殊符号
	- 重定向符号`>` `<`



### 补：==Linux命令中的通配符==
- 什么是通配符
	- 是一种用于匹配文件名或字符串的特殊字符或模式
	- 允许你根据一定的模式来匹配文件名或字符串，而不需要提供完整的准确名称
- Linux中常用通配符
	- 问号（`？`）：问号通配符匹配单个任意字符。如`f?ag`，可以是`faag`也可以是`flag`
	- 星号（`*`）：匹配零个或多个任意字符。如*.txt，匹配所有txt结尾的字符
	- 方括号（`[]`）：匹配指定范围内的单个字符或字符集合中的任意一个字符
		如`flag[12].txt`匹配`flag1.txt`和`flag2,php`
	- 花括号（`{}`）：表示可选的字符串。如`flag{12，13}`，表示flag12和flag13









