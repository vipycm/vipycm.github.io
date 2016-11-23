---
layout: "post"
title: "修改adb shell里面的终端窗口宽度"
date: "2016-11-22 12:30"
categories:
- Android
tags:
- adb
---
在adb shell 中，如果进入到一个很长或者很深的目录之后在输入命令时，命令会自动折行，并且折行后的命令还显示不全，这非常不利于命令输入，这里提供一种解决办法。<!--more-->
首先得安装busybox，安装方法参考：  
http://www.cnblogs.com/xiaowenji/archive/2011/03/12/1982309.html
安装好busybox之后进入adb shell，执行以下命令可将终端大小设置为当前窗口大小：

```
busybox resize
```
![adbshellresize](/images/2016/11/adbshellresize.png)

图中的红色竖线时默认的宽度，执行“mkdir longlonglong：时被自动折行，并且折行后mkdir的第一个字母"m"看不见了。执行resize之后突破了这个宽度，输入的长命令也能够完整显示。
