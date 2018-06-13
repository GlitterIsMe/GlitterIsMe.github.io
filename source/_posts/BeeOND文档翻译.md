---
title: BeeOND文档翻译
date: 2018-04-28 11:15:03
tags: BeeOND
categories: Tech
---
# BeeOND：BeeGFS On Demand
现在，集群的计算节点通常配备有内部的flash设备来存储操作系统并为应用提供临时的数据存储。但是使用临时的数据存储常常对于分布式的应用来说并不方便或者说并不实用，因为它们往往需要访问在不同节点上的共享的数据，这就导致SSD的高带宽以及IOPS被浪费。
![BeeOND结构示意图](http://p8701ip7e.bkt.clouddn.com/beeond.png)
<!-- more -->
## BeeOND精妙地释放了计算节点内部的存储设备的潜在性能
BeeOND通过允许在这样的内部存储设备上为计算任务创建**一个共享的并行文件系统**来解决这个问题。BeeOND实例只存在于任务运行期间以及只存在于它们被分配的节点上。这就为任务提供了快速的、共享的全闪存文件系统，作为一个精妙的Burst Buffer或者一个完美的用于存储临时数据的地方。这可以用来消除许多令人讨厌的I/O访问，这些访问会触发全局文件系统的随机访问。
![](D:\BurstBuffer\BeeGFS\BeeOND_pic\beeond.png)
BeeOND基于一般的BeeGFS服务，这意味计算节点将也会裕兴BeeGFS storage以及metadata服务来提供对这些节点内部存储设备的共享的访问。因为BeeOND不是仅仅访问内部的存储设备而是仅仅将数据存储到内部存储设备的一个子目录下，所以内部存储设备仍然可以由应用作为直接的本地目录来访问。
## 即使Global Cluster Filesystem并不基于GFS也可以使用BeeOND
虽然推荐将BeeOND与一个全局的BeeGFS文件系统组合使用，但是**无论这个全局共享集群文件系统是否基于BeeGFS，BeeOND都可以独立工作**。BeeOND仅为计算任务创建一个新的挂载点。任何标准工具都可以用于将数据转入或者转出BeeOND，而BeeOND包也包含了一个并行的copy工具用于在BeeOND实例与其他文件系统之间（比如持久化的全局BeeGFS）传输数据。
BeeOND实例可以通过一条简单的指令被创建或者销毁，可以很容易地被集成到cluster batch system的prolog以及epilog script中，比如Torque、Slurm或者Univa Grid Engine
