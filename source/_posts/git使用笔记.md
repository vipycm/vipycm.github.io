---
layout: "post"
title: "git使用笔记"
date: "2017-01-05 17:28"
categories:
- Tools
tags:
- git
---
## 记住用户名/密码
```
git config credential.helper store
```

##  忽略已被提交的文件

#### 方法一：暂时忽略，当文件被团队其他人修改并push后会失效

- 将指定的PATH加入忽略列表
```
git update-index --assume-unchanged <PATH>
```

- 取消忽略PATH
```
git update-index --no-assume-unchanged <PATH>
```
<!--more-->
- 查看已被忽略的列表
```
git ls-files -v | grep '^h'
```

#### 方法二：永久忽略，即使被团队其他人修改并push后依然生效

- 将指定的PATH加入忽略列表
```
git update-index --skip-worktree <PATH>
```

- 取消忽略PATH
```
git update-index --no-skip-worktree <PATH>
```

- 查看已被忽略的列表
```
git ls-files -v | grep '^S'
```
