---
title: BeeOND安装与使用
date: 2018-05-02 19:37:01
tags: BeeOND
categories: Tech
---
BeeGFS官网关于BeeOND的安装与使用方法简介
<!-- more -->
# Installation
BeeOND作为一个标准包可以通过BeeGFS仓库获取，并且可以通过直接通过包管理器安装（比如RedHat中的yum）

对于操作，**BeeOND需要BeeGFS的server以及client部件**。因此相应的包将会作为依赖包并自动安装。BeeOND package需要在所有BeeOND实例的主机上安装。
如果需要在BeeOND环境使用RDMA，那么你需要在client模块允许它，这可以通过编辑*/etc/beegfs/beegfs-client-sutobuild.conf*实现（详细见client内核模块编译）

---
# Usage
**BeeOND包的主要组成部分是一个启动与停止BeeGFS文件系统实例的脚本**，这个脚本在*/opt/beegfs/sbin/beeond*。BeeOND instance能够通过`beeond start`以及`beeond stop`来控制
通过下面的一系列参数来运行一个beeOND实例：
`$ beeond start -n nodefile -d /data/beeond -c /mnt/beeond`
参数:
- nodefile：包含所有运行BeeOND实例主机名的文件，所有在这个文件上的主机都会成为这个新的文件系统实例的client以及server
- /data/beeond：BeeOND实例在每个节点上存储原始数据的路径
- /mnt/beeond：BeeOND实例在每个节点的挂载点

通过脚本中的`stop`指令来管理BeeOND实例：
`$ beeond stop -n nodefile -L -d`
参数:
- nodeile：所有在启动的时候使用的主机名
- -L：删除所有BeeOND实例生成的log文件
- -d：删除所有BeeOND实例写的原始数据

更多相关信息：
`$  beeond -h`

---
# Additional Tools
## beegfs-ctl
使用：添加`--mount`参数
作用：列出BeeOND实例中注册的存储节点
示例：
`$ beegfs-ctl --mount=/mnt/beeond --listnodes --nodetype=storage`

## beegfs-cp
作用：将数据拷入或者拷出BeeOND
示例：
`$ beegfs-cp copy -n nodefile /projects/dataset01 /projects/dataset02 /mnt/beeond` 

参数：
- nodefile：包含所有主机名
- /projects/dataset01 & /projects/dataset02：要拷贝数据的源路径（支持一个或者多个源文件或者源目录）
- /mnt/beeond：目标路径
---
> https://www.beegfs.io/wiki/BeeOND

