---
title: adb shell 常用命令
date: 2016-11-13 18:15:47
categories:
- Android
tags:
- adb
---
1. 查看最上层成activity名字
``` bash
#linux:
adb shell dumpsys activity | grep "mFocusedActivity"
#windows:
adb shell dumpsys activity | findstr "mFocusedActivity"
```
    <!--more-->
2. 查看service列表
``` bash
adb shell dumpsys activity services [包名（可选）]
```
3. 查看window列表
``` bash
adb shell dumpsys window
```
4. 启动activity
``` bash
adb shell am start -n <包名/完整类名>  --es extra_key "extra_value"
#（-n 类名,-a action,-d date,-m MIME-TYPE,-c category,-e 扩展数据,等）
#扩展数据类型：
#-e String, int
#--es String
#--ez boolean
#--ei int
```
5. 启动service
``` bash
adb shell am startservice -n <包名/完整类名>
#同启动activity一样也可以添加扩展数据
```
6. 挂载/system 为可读写
``` bash
adb shell
su
mount -o remount,rw /system
```
7. 查看ip等网络信息
``` bash
adb shell ifconfig
```
8. kill adb服务
有时候adb shel找不到设备，可以先杀掉adb服务再试试
``` bash
adb kill-server
```
9. kill adb进程
有时候杀掉adb服务还是不行，可以尝试杀掉adb进程
``` bash
#linux
killall adb
#windows
taskkill /f /im adb.exe
```
10. 安装apk
``` bash
adb install <apkfile>
```
11. 卸载已安装的app
``` bash
adb uninstall <package>
```
