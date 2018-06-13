---
title: TiDB学习
date: 2018-06-05 16:25:51
tags: [TiDB, TiKV]
categories: Tech
---

# TiDB简介与整体架构
## 简介
TiDB是一个开源分布式HTAP数据库（Hybrid Transactional and Analytical Processing），主要特点是结合了传统的RDBMS以及NoSQL。目标是为OLTP以及OLAP场景提供一站式解决方案。
特性：
- 兼容MySQL：无需修改代码直接从MySQL迁移到TiDB
- 支持水平无限扩展：增加节点即可实现扩展
- 强一致性：标准ACID事务
- 高可用：基于Raft的多数派选举协议保证数据强一致性 & 自动故障恢复

<!-- more -->
## 整体架构
![](https://ws1.sinaimg.cn/large/8c185877gy1fs0dqkqnhlj20ge0c0mxu.jpg)

- TiDB Server
	- 接收SQL请求，处理SQL逻辑
	- 通过PD找到数据所在TiKV地址，并获取数据
	- 不存储数据，无限水平扩展
- PD Server(Placement Driver)
	- 存储集群元数据（kv-TiKV节点的映射）
	- TiKV集群的调度和负载均衡
	- 分配全局唯一的事务ID

![](https://ws1.sinaimg.cn/large/8c185877gy1fs0bkxz9vgj20mu0ahdgm.jpg)

- TiKV Server
	- 分布式KV存储引擎
	- 以region为单位存储kv数据，每个region对应一个key-range，每个节点负责多个region
	- 基于raft协议的副本管理，基本单位为regin，不同节点的多个region构成一个raft group，group内的region互为副本


## TiKV存储技术
存储栈示意图：
![](https://ws1.sinaimg.cn/large/8c185877gy1fs0b09hsloj20k705n3z3.jpg)

*整体使用rust语言开发*
- RocksDB
使用RocksDB作为本地存储引擎存储KV数据
- Raft
Raft协议是一个一致性协议，通过Raft可以保证安全的数据拷贝。数据写入是通过Raft层接口实现。
- Region
TiKV中对Key的划分是按照key range进行，将一段key range对应一个服务器，并且叫做一个region。region以256MB为单位，保存[start_key, end_key]作为region的元数据，数据以region为单位分布并且以region为单位做raft复制以及成员管理。
![](https://ws1.sinaimg.cn/large/8c185877gy1fs0bdrg95ej20nt0ct3zc.jpg)

- MVCC
TiKV的多版本控制，解决锁开销，多个client写同一个key不加锁，而是作为两个不同的version来写
- Transaction
基于Google的Percolater模型并进行优化，采用乐观锁，不检测写写冲突，而是当发生冲突的时候早提交的会成功，失败方重新执行事务。当冲突严重的时候性能变差。

## TiDB的SQL层实现
### 对SQL的需求
- table的元信息存储
- row的存储：按行存储
- index的支持
	- 支持Primary Index以及Secondary Index
	- 支持点查以及范围查询
	- 支持Unique Index以及非Unique Index（索引项没有重复即课创建Unique Index）
	- 支持基本操作inset、update、dalete、select

### SQL到Key-Value的映射
1. 每个表分配TableID，每个索引分配IndexID，每一行分配RowID（TableID集群内唯一，IndexID与RowID表内唯一），以此构成KV数据（如果row有int类型的primary key，那么就把这个作为RowID）：
```
key: tableprefix_rowprefix_tableID_rwoID
value: [col1, col2, col3, col4]
```
对于索引：
```
key: tableprefix_idxprefix_tableID_indexID_indexColumnsValue
value: rowID
```
对于非Unique Index，可能存在相同的indexColumnsValue，所以对非Unique索引：
```
key: tableprefix_idprefix_tableID_indexID_ColumnsValue_rowID
value: null
```
*注：xxprefix都是常量*
2. key的编码
主要注意的问题：通过设计使得编码前后的key比较关系不变(Memcomparable)，也就是说编码前，两个数据的值的大小关系与编码后一致（采用codec编码）
3. 示例

|name|descri|value|
|-----|-----------|---|
|TiDB|SQL Layer|10|
|TiKV|KV Engine|20|
|PD|Manager|30|

KV数据：

|key|value|
|----|------|
|t_r_10_1|[TiDB, SQL Layer, 10]|
|t_r_10_2|[TiKV, KV Engine, 20]|
|t_r_10_3|[PD, Manager, 30]|

index数据（假设indexID为1）：

|key|value|
|----|------|
|t_r_10_1_10_1|null|
|t_r_10_1_20_2|null|
|t_r_10_1_30_3|null|

### 元信息管理
元信息：Database以及Table的元信息
编码：分配唯一的Database ID或者TableID，加上前缀m_；value为序列化之后的元信息
Schema信息：使用一个专门的KV存储（有什么用），基于F1算法的无损Schema变更

### SQL on KV
架构图
![](https://ws1.sinaimg.cn/large/8c185877gy1fs033rkk3dj20rw0brt94.jpg)

TiKV Cluster：KV存储引擎
TiDB server：SQL层，不存储数据，节点之间对等，处理用户请求，进行SQL运算

### SQL运算
SQL layer总体结构：
![](https://ws1.sinaimg.cn/large/8c185877gy1fs0bpmaktej20ny09y77b.jpg)

用户请求下发过程：
1. Load Balancer获取请求，发送到tidb-sever
2. tidb-server解析MySQL Protocol Packet获取请求内容
3. tidb-server进行一系列检查、优化处理数据，并与tikv交互获取数据
4. 返回数据、


为了减少过多的运算所做的优化：
- 计算尽量靠近存储节点，减少RPC(Remote Procedure Call))调用
- filter尽量靠近存储节点，只返回有效的行
- 聚合函数、Groupby下推到存储节点，进行预聚合

示例：
count：
![](https://ws1.sinaimg.cn/large/8c185877gy1fs0c5laefuj20kw0afwfe.jpg)
将count下发到各个TiKV server中，filter在各个TiKV内部进行，TiDB统计返回的结果

join：
```
CRAETE TABLE t1(id INT, email TEXT, KEY idx_id(id));
CREATE TABLE t2(id INT, email TEXT, KEY idx_id(id));

SELECT * FROM t1 JOIN t2 WHERE t1.id = t2.id
```
支持的join操作类型：hash join；sort merge join；index-lookup join
还做了其他的多种优化

## TiDB的调度
### 调度要解决的问题
- 副本数量不能多也不能少
- 副本需要分布在不同的机器上
- 新加节点后，可以将其他节点上的副本迁移过来
- 节点下线后，需要将该节点的数据迁移走
- leader分布均匀
- 节点存储容量均匀
- 热点访问均匀
- 减少负载均衡对在线服务的影响
- 节点状态的管理

### 基本操作
- 增加一个replica
- 删除一个replica
- 切换leader

Raft协议中通过AddReplica、RemoveReplica、TransferLeader可以实现

### 信息收集
TiKV定期向PD汇报两类信息：
- 整体信息
通过TiVK节点与PD间的心跳包，既能够检测TiKV是否存活，同时也通过心跳包获取TiKV store的状态信息
- Raft group信息
每个Raft Group与PD间的心跳包，汇报Region的状态（Leader位置、Follower位置、掉线Replica个数、数据读写速度）

PD根据心跳包反馈的信息进行决策判断节点的状态，如心跳包中断一定时长则判断某个节点下线，则将这个store上的region都调度走。或者可以通过PD接口手动下线一个store。

### 调度策略
1. 确保replica数量正确
通过心跳包发现replica数量不符合要求的时候，通过Add或者Remove调整replica的数量，造成replica数量变动的原因比如：节点掉线replica减少、掉线节点恢复replica增加、手动增加
2. 确保一个Raft Group的replica在**不同的位置**
这里不同的位置区别于不同的节点，即多个不同的节点可能位于同一个位置。为每个节点配置一个location-label，在分配节点的时候尽量保证replica处于不同的label上
3. 副本在store之间均匀分布
由于每个副本数据容量上限固定，所以只需要维持节点上副本数量均衡
4. leader的均匀分布
将leader在节点中分开
5. 热点数据均匀分布
检测leader上报信息中的负载信息，识别热点并将其分散
6. store存储空间均衡
store的存储空间有一定上限，PD调度的时候会考虑
7. 避免负载均衡影响服务响应速度
PD会限制当前正在进行的操作数量
8. 手动操作上下线节点
pd-ctl

### 总体流程
PD接收store或者raft leader的心跳包判断其运行状态，并根据该状态生成操作序列。对于region leader则会检查是否需要进行操作，如果需要则通过心跳包发送建议，region leader收到之后自行决定执行与否以及时机

## TiDB工具
### Syncer
作用：数据导入工具，实时从MySQL迁移数据
![](https://ws1.sinaimg.cn/large/8c185877gy1fs0cmgwzmfj20gm07xaak.jpg)

### TiDB-Binlog
手机TiDB的Binlog，提供实时备份与同步功能。
- 数据同步：同步TiDB的数据到其他数据库
- 实时备份和恢复：备份TiDB的集群数据， 并且可以用于集群故障时恢复
![](https://ws1.sinaimg.cn/large/8c185877gy1fs0cmgrlz8j20ks082jru.jpg)

主要部件功能：
- Pump：守护进程，实时记录TiDB产生的Binlog并顺序写入Kafka中
- Drainer：从Kafka中手机Binlog，按照TiDB中的提交顺序转化为指定数据库支持的SQl语句并进行同步
- Kafka & ZooKeeper：存储Binlog数据并提供给Drainer
注：*Kafka是一个分布式发布订阅消息系统*
### MyDumper/MyLoader
并行地备份或者恢复

## 参考
> https://pingcap.com/docs-cn/
> https://github.com/pingcap/tikv
> https://github.com/pingcap/tidb


