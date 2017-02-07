layout: "post"
title: "Screen overlay 问题排查"
date: "2017-02-07 16:42"
categories:
- Android
tags:
- ScreenOverlay
- dumpsys
---
Android6.0在授权的时候如果在设置界面有其他窗口显示，会出现screen overlay的警告无法授权，然后会去引导关闭app的顶层窗口显示，但是由于TOAST_WINDOW类型的window则无需允许顶层窗口的权限就能显示在最顶层，这就导致了因为某些app显示了TOAST_WINDOW类型的window而无法授权。要想授权就只能找到是哪个app显示了顶层window，卸载之后就能正常授权了。
<!--more-->
1. dumpsys window
```
adb shell dumpsys window
```
2. 找到WINDOW MANAGER WINDOWS,看Surface是否shown
![method](/images/2017/02/dumpsys_window.png)
3. 卸载package对应的app之后再次授权
