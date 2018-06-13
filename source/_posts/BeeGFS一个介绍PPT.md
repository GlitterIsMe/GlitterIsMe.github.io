---
title: BeeGFS一个介绍PPT
date: 2018-05-03 20:37:40
tags: BeeGFS
categories: Tech
---

# BeeGFS Solid, fast, and made in Europe
## What is BeeGFS
BeeGFS是一个独立于硬件的并行文件系统，为对性能要求较高的环境设计的
从图中可以看出，写到beegfs目录中的文件被条带化存储到多个Storage Server中，同时将元数据存储到MetaData Server
<!-- more -->
---
## BeeGFS Architecture
- **client**：挂载文件系统的linux模块
- **Storage Server**：存储文件数据
- **Metadata Server**：维护文件条带的信息，在open-close之间不会被访问*（open/close获取元数据之后就是与Storage Server的数据传输）*
- Management Service：聚集所有的部件以及监测服务
- Graphical Adiministration and Monitoring System：图形监视

---
## Key Aspect
### Performance & Scalability
- 为HPC优化
- 完全的多线程、轻量化设计
- 支持RDMA/RoCE以及TCP
- 分布式文件数据：多个server聚集的带宽
- 跨多个server的元数据分布
- 单文件流性能极高（多个GB/s）
### Flexibility
- 多个服务可以运行到同一个机器
- 灵活的条带化选择：per-file或者per-dictory
- 可以直接在运行时添加server
- BeeOND
- 可在多个Linux版本运行
- 可运行到不同的处理器架构上
- NFS/ SMB/CIFS re-export possible
### Robust & easy to use
- 应用可以像访问一般的文件系统挂载点一样访问BeeGFS
- 系统指令集运行在用户空间，在标准本地文件系统（ext4、xfs、zfs）之上
- 没有客户端内核补丁，内核升级很简单
- 提供RedHat、SUSE、Debia以及其衍生版的包
- 独立于硬件
- 图形化的监测工具

---
## BeeOND
- 可以直接创建一个并行文件系统实例
- 通过单条指令启动或者停止
- 使用案例：云计算、测试系统、集群计算节点
- 可被集成到集群批处理系统*cluster batch system*
- 一般的应用场景：”per-job parallel filesystem“（单任务的并行文件系统）
	- 聚集一个任务的计算节点的本地的SSD/硬盘的性能与容量
	- 从全局存储加载数据
	- 针对各种IO访问模式的加速 	
