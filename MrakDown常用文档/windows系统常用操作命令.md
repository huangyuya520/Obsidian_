F### 1.设置本地密码安全策略

**在"运行"窗口打开本地安全策略 secpol.msc** 

### 2.常用命令 

  **查看当前用户 net user**
  **查看用户的属性 net user <用户名>**
  **查看本地用户 net localgroup**
  **新建一个用户   net user `<username>` `<password>` /add 或者 net user `<username>` `<password>` /ad**
  **把用户加到管理组 net localgroup Administrators `<username>` /add**
  **用`&&`连续执行命令(新建用户并将这个用户加入到管理员组中)**
  **net user `<username>` `<password>` /add && net localgroup Administrator**
**s `<username>` /add**
  **删除账号命令 net user moon /del**
  **查看本地密码安全策略 net accounts**
  查看当前会话 net session
  **设置python虚拟环境       python -m venv `<vnevname>`**
  **激活python虚拟环境      windowos:   `<venvname>`\Scripts\Activate.ps1**
					  **Linux:   source `<venvname>`/bin/activate**


  **ipconfig /flushdns 刷新DNS缓存**
  **netstat -ano**   查看所有开放端口及其相关进程
#### python包升级命令
  **pip list --outdated        使用pip检查哪些包需要更新**
  **pip install --upgrade <要升级的包名>        使用pip升级python包**
  **pip-review --interactive      使用python的第三方pip-review包实现对python包的更新**




### 3.用户相关

  **查看当前用户 whoami** 
  **查看当前用户特权信息 whoami /priv**
  **查看当前的登录账号 query user** 

### 4.远程访问 

  **打开远程桌面  mstsc** 

### 5.网络相关命令 

  查看netstat 有那些参数    netstat /? 
  **查看本地和外部的    ip和端口    netstat -ano** 
  **查看网卡信息    ipconfig** 
  **查看网卡信息和dns    ipconfig /all** 
  **判断主机是否存活    ping ip或主机名           icmp协议去访问目标地址的  TTL** 
  **强制返回ipv4 地址 ping -4 `<hostname>`**
  **查看路由表    route print**
  **查看arp信息    arp -a**
  
  ==**重新获得IPV6地址     ipconfig /renew6 WLAN**==




### 7.系统信息

   **查看主机名  hostname**
   **查看信息信息 systeminfo**    包括主机名 os版本信息 cpu硬件信息 补丁信息

### 8.进程相关的命令

   **查看进程的 tasklist**
   **显示详细任务信息 tasklist /V**
   列出模块 tasklist /M
   列出任务的详细信息 TASKLIST /V /FO CSV 
   列出详细的关务信息 导出到文件里TASKLIST /V /FO CSV > c:\task.csv 
   **强制结束进程  taskkill /f /t /im <进程名>**
                **taskkill /pid <PID会话名> -t -f**

   **寻找指定进程的pid tasklist |findstr "<进程名>"** 
           强制关闭 taskkill /pid <PID会话名> -t -f

### 9.操作命令 

  **列出当前路径所有目录和文件 dir**
  列举出当前目录隐藏文件和系统文件  dir /a
  显示当前目录文件所有者信息   dir /q
  显示指定目录和所有子目录中的文件  dir /s
  使用小写显示当前目录未排序的目录名和文件名  dir /l
  **进入某个目录 cd** 
  **跳到上级目录 cd ..**
  **进入某个盘 d:**
  **强制进入某个目录  cd /d 路径**
  **创建写入文件 echo <内容> > <文件名>**
  **查看文件 type <文件名>**
  **复制文件 copy <文件名> <新的文件名>**
  **移动文件  move <旧文件> <新文件>**
  **删除文件 del <文件名>**
  **创建文件夹 mkdir <文件夹名>**
  **搜索文件命令 进入到某个盘搜索文件  cd /d c:\ && dir /b /s <文件名>**





### 10.注册表

   查看远程终端的开发端口 REG query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v PortNumber   
返回16进制 转 10进制 

### 11.ipc链接 

   **查看电脑中共享的资源 net share**
   **建立ipc链接  net use \\\\`<hostIP>`\ipc\$ "`<password>`" /user:"`<username>`"**
   **访问ipc链接资源  dir \\\\`<hostIP>`\c\$**
   **查看当前ipc链接  net use**
   **关闭ipc连接 net use \\\\`<hostIP>` /del**


### 12 .域命令

   **查看当前用户权限: whoami /user**

可知域名为和其他信息: net config workstation

**查询域用户：net user /domain**

**添加域用户: net user 用户名 密码 /add /domain**

**添加域管理员: net group "domain admins" 用户名 /add /domain**

添加企业管理员: net group "enterprise admins" 用户名 /add /domain

查询域管理员用户：net group "domain admins" /domain

查询域企业管理组: net group "enterprise admins" /domain

查询域本地管理组: net localgroup administrators /domain

查询域控制器和时间：net time /domain

查询域名称：net view /domain

查询域内计算机：net view /domain:redteam.local

查看当前域内计算机列表: net group "domain computers" /domain

查看域控机器名: net group "domain controllers" /domain

查看域密码策略: net accounts /domain

查看域信任: nltest /domain_trusts

查看某个域的域信任: nltest /domain_trusts /all_trusts /v /server:ip

### 13.其他命令
%localappdata%  打开系统存储本地缓存的默认路径