layout: "post"
title: "samba安装及配置"
date: "2017-02-09 12:03"
categories:
- Tools
tags:
- samba
---
为了实现Windows主机与Linux服务器之间的资源共享,可以在Linux上搭建一个samba服务器，然后其他机器就可以通过 \\\\ip 的方式来读写共享的目录，实现文件共享。

1. 安装samba
```
sudo apt install samba
```
    <!--more-->
2. 配置
```
sudo vim /etc/samba/smb.conf

#在smb.conf文件最后增加以下配置
[share]
    comment = Ubuntu File Server Share
    path = /home/mao/Share/ #共享的目录
    writable = yes
    public = yes #无需用户名密码就可以访问
    forceuser = mao #系统用户名，这样别的机器访问的时候也能添加或删除文件
```
3. 启动、停止、重启命令
```
sudo service smbd start
sudo service smbd stop
sudo service smbd restart
```
