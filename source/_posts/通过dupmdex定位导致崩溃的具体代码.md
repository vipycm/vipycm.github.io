layout: "post"
title: "通过dupmdex定位导致崩溃的具体代码"
date: "2016-12-10 17:52"
categories:
- Android
tags:
- dexdump
- crash
---

![crash](/images/2016/12/crash.png)

因为打包时的优化，导致堆栈中记录的行号和源代码对不上（源代码SpaceDataScan中根本就没有36248行），所以这里通过dumpdex的方式找到对应的代码行号
<!--more-->
## 解压apk文件
```
unzip xxx.apk -d crash
cd crash
```

## 查找SpaceDataScan这个类在哪一个dex文件中
```
grep "SpaceDataScan" . -rn
#查询结果：Binary file ./classes3.dex matches
```
说明这个类在classes3.dex中，那么我们就需要dumpdex

## dexdump
```
$ANDROID_HOME/build-tools/23.0.1/dexdump -hdf ./classes3.dex > classes3
```

## 用vim打开dump出来的这个文件
```
vim classes3
```

## 搜索崩溃的类定义(Class)

```
/Class descriptor  : 'Lcom\/cleanmaster\/ui\/space\/scan\/b;
```
![class](/images/2016/12/class.png)

## 在这个类定义后面搜索找到对应的方法定义
```
/name          : 'f'
```
![method](/images/2016/12/method.png)

## 在这个方法后面搜索对应的崩溃行号
```
/36248
```
![line](/images/2016/12/line.png)

行号前面的0x0110代表执行这一行代码的具体偏移地址，从0x0110到0x0138都是36248这一行代码要执行的内容，在这之间产生的崩溃都会上报到36248这一行

## 向上翻找到0x0110
![inline](/images/2016/12/inline.png)

从崩溃堆栈可以看到，崩溃的地方时执行了list.get(9), 然后在上图中找到0124到0128正是在调用这个方法，这个list是Lcom/cleanmaster/ui/space/newitem/q的一个静态变量a，通过打包时产生的mapping文件找到Lcom/cleanmaster/ui/space/newitem/q这个类，然后找到静态变量a，最后发现这个list里面一共只有9个元素，所以调用list.get(9)时导致崩溃。
