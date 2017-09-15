layout: "post"
title: "使用本地maven仓库"
date: "2017-09-15 15:48"
categories:
- Tools
tags:
- maven
---

1. 下载 [maven](http://maven.apache.org/download.cgi)

2. 解压并配置环境变量
```
# maven
export M2_HOME=/home/mao/Programs/apache-maven-3.5.0
export PATH=$PATH:$M2_HOME/bin
```
3. 配置gradle
```
maven { url 'file:///home/mao/.m2/repository' }
```
4. 向本地仓库添加依赖
```
mvn install:install-file -Dfile=jar包的位置 -DgroupId=groupId -DartifactId=artifactId -Dversion=version -Dpackaging=jar/aar
```
    参考[http://www.blogjava.net/fancydeepin/archive/2012/06/12/380605.html](http://www.blogjava.net/fancydeepin/archive/2012/06/12/380605.html)
