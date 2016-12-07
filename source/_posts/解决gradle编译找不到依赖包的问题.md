layout: "post"
title: "解决gradle编译找不到依赖包的问题"
date: "2016-12-07 12:09"
categories:
- Android
tags:
- gradle
---
所有配置都正确，在别人的机器上都能编译通过，但是在我的机器上就是不编译不过，错误信息如下：
```
Error:A problem occurred configuring root project 'xxx'.
> Could not find support-v4.jar (com.android.support:support-v4:24.1.1).
  Searched in the following locations:
      https://jcenter.bintray.com/com/android/support/support-v4/24.1.1/support-v4-24.1.1.jar
```
<!--more-->
## 原因：
猜测是因为gradle在本地缓存的依赖包出错导致的
## 解决方案：  
关闭androidstudio，直接删掉gradle的所有cache文件: C:\Users\USERNAME\.gradle\caches

删除之后重新编译就能够成功了。

由于需要重新下载所有的依赖包，这会浪费很多时间，一个更优雅的解决方案就是在caches下找到出错的依赖包的目录并删掉，然后再重新打包（未验证）
