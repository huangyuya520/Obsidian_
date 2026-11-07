### 1. 查看与连接设备


```

adb devices               查看已连接设备


adb devices -l           显示更详细的设备信息


adb start-server       启动 ADB 服务


adb kill-server          停止 ADB 服务


adb kill-server
adb start-server       重启 ADB 服务


```

常见设备状态：

| 状态             | 含义                 |
| -------------- | ------------------ |
| `device`       | 设备已正常连接            |
| `unauthorized` | 手机尚未允许 USB 调试授权    |
| `offline`      | 设备已连接，但通信异常        |
| `无设备`          | 驱动、数据线或 USB 调试存在问题 |

### 2. 无线连接设备
==手机和电脑需要处于同一网络==
```
# 先通过 USB 连接手机，然后开启 TCP 模式
adb tcpip 5555

# 查看手机 IP 地址
adb shell ip addr show <wlan0>

# 连接设备
adb connect <192.168.1.100:5555>

# 查看连接结果
adb devices

# 断开指定设备
adb disconnect <192.168.1.100:5555>

# 断开所有无线设备
adb disconnect

adb usb      恢复USB模式
```

### 3. 进入 Android Shell
```
# 进入设备终端
adb shell

# 直接执行一条命令
adb shell <ls /sdcard>

# 查看当前用户
adb shell whoami

# 查看系统属性
adb shell getprop

# 查看 Android 版本
adb shell getprop ro.build.version.release

# 查看 SDK 版本
adb shell getprop ro.build.version.sdk

# 查看设备型号
adb shell getprop ro.product.model

# 查看设备品牌
adb shell getprop ro.product.brand

# 查看设备 CPU 架构
adb shell getprop ro.product.cpu.abi

exit    退出Shell
```

### 4. 安装与卸载 APK
(略)


### 5. 查询应用包名
```
# 查看所有应用包名
adb shell pm list packages

# 查看第三方应用
adb shell pm list packages -3

# 查看系统应用
adb shell pm list packages -s

# 搜索包含指定关键词的包名
adb shell pm list packages | grep <packages name>

```


### 6. 启动和停止应用
(略)


### 7
(略)
### 8
(略)

### 9
(略)

### 10. 查看日志
```
abd logcat      实时查看日志
adb logcat -c   清空日志
adb logcat *:E   只查看错误日志
```








































