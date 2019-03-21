---
title: "Linux系统限制 - ulimit"
permalink: /domains/devops-ulimit/
excerpt: "Linux系统限制 - ulimit"
last_modified_at: 2018-11-25T22:21:33-05:00
redirect_from:
  - /theme-setup/
toc: true
---

### 背景故事
几年前在作者所在的项目中的生产环境曾经发生了一个情况，我们20多个微服务其中一个JAVA服务在上线初始运行良好，但过了一天或两天就出现不能响应请求的情况，日志中出现"Resource temporarily unavailable"的错误。而当时的系统资源并未出现过载的情况。查到最后发现运行该JAVA程序的用户对应的最大线程ulimit设置过小，服务线程还没来得及释放导致线程堆积，那么针对用户最大线程应做相应的优化。

### ulimit命令详解
ulimit为Linux系统内核参数, 其作用主要是限制每个用户所能够使用的设备资源情况，在某用户被创建时Linux系统就为此设置了用户ulimit的默认值，如下为RHEL6系统默认设置(不同Linux发布版不尽相同):

![ulimit]({{ "/assets/domains/ulimit.png" | relative_url }})

```
ulimit [-SHacdefilmnpqrstuvx]
参数S：表示软限制，当超过限制值会报警
参数H：表示硬限制，必定不能超过限制值
参数t：每个进程可以使用CPU的最大时间
参数f：当前shell可创建的最大文件容量
参数d：每个进程数据段的最大值
参数s：堆栈的最大值
参数c：当某些程序发生错误时，系统可能会将该程序在内存中的信息写成文件(除错用)，这种文件就被称为核心文件(core file)。此为限制每个核心文件的最大容量
参数m：可以使用的常驻内存的最大值
参数u：每个用户运行的最大进程并发数
参数n：每个进程可以同时打开的最大文件句柄数
参数l：可以锁定的物理内存的最大值
参数v：当前shell可使用的最大虚拟内存
参数a：将列出所有资源限制，如：
参数p：管道的最大值
```

> 使用`ulimit -u <线程数>` 修改当前用户线程数时应先su到相应用户下

### 使ulimit修改永久生效
修改`/etc/security/limits.conf`文件，`*`指所有用户的默认limit, 如下图52-55行定义了该`propel`账户在运行时的limit.

```
-bash-4.1$ vim /etc/security/limits.conf
 42 #*               soft    core            0
 43 #*               hard    rss             10000
 44 #@student        hard    nproc           20
 45 #@faculty        soft    nproc           20
 46 #@faculty        hard    nproc           50
 47 #ftp             hard    nproc           0
 48 #@student        -       maxlogins       4
 49 
 50 # End of file
 51 # Modified by propel_ha cookbook
 52 propel          soft    nproc   10000
 53 propel          hard    nproc   10000
 54 propel          soft    nofile  10000
 55 propel          hard    nofile  10000

 -bash-4.1$ vim /etc/security/limits.d/90-nproc.conf
 1 propel          soft    nproc   10000
 2 propel          hard    nproc   10000
```

