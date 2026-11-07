### 查看命令
**uname -m   查看系统架构**
**cat /etc/centos-release   查看系统版本**
**fastfetch -c all   显示获取系统详细信息**
ls /etc/yum.repos.d  查看自己的系统当前使用的镜像源
**yum repolist  查看已配置的软件仓库**
**lsb_release -a   显示当前Linux发行版的名称、版本号、操作系统版本以及其他相关信息**
**cat /etc/passwd 查看用户**
**cat /etc/group 查看所有用户组**


### 防火墙命令
**systemctl status firewalld   验证防火墙是否已启动**
**sudo systemctl stop firewalld    临时停止防火墙**
firewall-cmd --reload    重新加载防火墙规则
**sudo systemctl restart firewalld  重启防火墙**
**sudo firewall-cmd --permanent --add-service=<协议>    添加永久规则**
firewall-cmd --add-service=<协议名> --permanent 添加永久规则
**firewall-cmd --list-all  查看防火墙规则**
**sudo firewall-cmd --list-services  查看防火墙开放的服务**

