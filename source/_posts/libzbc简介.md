---
title: libzbc简介
date: 2018-05-11 10:44:58
tags: libzbc
categories: Tech
---
# libzbc and lkvs
## libzbc
### Overview
- libzbc库提供了接口用于操作支持Zoned Block Command(ZBC)以及Zoned device ATA Command set(ZAC)这两个指令集的设备
- libzbc基于INCITS技术协会T10和T13起草的ZBC/ZAC标准实现
- 允许应用访问ZBC HM磁盘
- 通过直接执行**SCSI(SG_IO)命令**实现访问zone的信息以及读写操作
- 为标准的SAS/SATA块设备上的操作提供ZBC模拟层（libzbc库内部会模拟磁盘的zone配置）
<div align=center>
![libzbc两种模式对比](https://ws1.sinaimg.cn/large/005xaSPRgy1fr76ar1pt4j30jr0890t5.jpg)
</div>

<!-- more -->

从图上可以看到，对于实际的ZBC磁盘，libzbc是直接通过SCSI命令实现访问磁盘的，跳过了VFS层以及块层。而在仿真模式，libzbc是通过系统调用read/write进行IO，此时会经过VFS层。

---
### libzbc API
**Main：**
<div align=center>
![libzbc interface](https://ws1.sinaimg.cn/large/005xaSPRgy1fr76j5inh9j30kz0cjjt0.jpg)
</div>
**Addition：**
这两个函数用于初始化**模拟ZBC设备**，zone的配置信息会在zbc_close中记录到磁盘
<div align=center>
  ![](https://ws1.sinaimg.cn/large/005xaSPRgy1fr76lah8ygj30l1049jrr.jpg)
</div>

---
## Linear Key Value Store Architecture
lkvs：一个简单的append-only的KV系统，通过libzbc查询磁盘信息以及读写数据
结构：
<div align=center>
![](https://ws1.sinaimg.cn/large/005xaSPRgy1fr76qb7qv5j30bu06b3yo.jpg)
</div>

*需要了解一下块设备文件的资料*

> SMR Impact on Linux Storage Subsystem, Jorge Campello, Adam Manzanares, HGST
> https://github.com/hgst/libzbc

