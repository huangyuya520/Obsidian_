**wsl --list --online    列出可用的 Linux 分发版**
**wsl --list --verbose    列出已安装的 Linux 分发版**
**wsl --set-default-version `<Version>`    设置默认 WSL 版本
wsl --update --web-download    更新 WSL**
**wsl --version          检查 WSL 版本**
**wsl --shutdown          立即终止所有正在运行的分发版和 WSL 2 轻型实用工具虚拟机**
**wsl --status             检查 WSL 状态**
**wsl -l -v                   检查wsl的运行状态与wsl对应的版本**
**ubuntu<版本号>.exe config --default-user `<username>`    切换wsl中系统的默认登录用户**

**taskkill /F /IM wsl.exe** 
**taskkill /F /IM wslservice.exe**
**taskkill /F /IM vmcompute.exe**       杀死wsl相关进程

