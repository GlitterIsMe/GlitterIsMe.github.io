---
title: BeeGFS代码结构
date: 2018-05-03 20:36:47
tags: BeeGFS
categories: Tech
---
BeeGFS代码目录整理（非完整）
<!-- more -->
# 代码目录
|目录|备注|
|-----|-----|
|auto_pacakage|安装相关|
|beeond|beeond相关脚本文件（start、end等）|
|beeond_thridparty|beeond第三方库|
|beeond_thirdparty_gpl|beeond第三方库|
|build|编译相关|
|client_devel||
|<font color=red>**client_module**</font>|客户端内核模块|
|client_tests|客户端测试|
|common||
|common_package||
|<font color=red>**ctl**</font>|beegfs-ctl工具|
|deeper_cached||
|deeper_common||
|deeper_lib||
|event_listener|事件监听？|
|fsck||
|<font color=red>**helperd**</font>|为client提供辅助功能(DNS, LOG等)|
|java_lib|java库|
|<font color=red>**meta**</font>|元数据相关|
|<font color=red>**mgmtd**</font>|一个集群系统工具？|
|mon||
|opentk_lib||
|<font color=red>**storage**</font>|存储相关|
|testing|系统测试|
|thridparty|第三方库|
|upgrade||
|utils||
|utils_devel||

