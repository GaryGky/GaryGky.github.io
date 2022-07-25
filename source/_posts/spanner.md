---
title: spanner
date: 2022-05-31 21:25:08
banner_img: /img/bg/sunset.jpeg
index_img: /img/cover/spanner.png
tags: [Distributed Storage, Google, NoSQL]
---

Spanner: Google’s Globally-Distributed Database
前言：个人感觉这篇paper组织的逻辑比 Dynamo和GFS 难理解一些，因为时而讲存储引擎层的内容，时而讲存储Service层的内容。笔者按照自己的逻辑重新组织了一下文章，希望能够帮助读者更好的理解Spanner。

推荐阅读：为了更好的体验，请移步至飞书 [Spanner](https://lo845xqmx7.feishu.cn/docs/doccnALxkfZlATXInaIF2BcyMUf).

特征
- 可扩展的
- 基于多版本读 (MVCC) 
- 基于两阶段锁的写 (2PL)
- 同步复制
- 提供 Externally Consistent
Spanner 简介
Spanner是一个基于Paxos的，支持数据分片的全球部署的强一致数据库。
即使在极端自然灾害下，Spanner能够对上层应用提供高可用的服务，因为Spanner的数据是跨大陆存储的。F1 Google 是Spanner最初的用户，它为实现HA在美国部署了5个分片。
大多数应用对于HA的要求基本是能够容忍1-2个DC断电，根据Paxos算法 (N = 2F + 1)，Spanner只需要提供 3-5 个服务节点就可以满足Fault Tolerance的需求。

Spanner 主要的关注点在于跨机房的数据同步。
- 对比Bigtable，Spanner能够提供DB级别更好的支持：
  - 支持更加复杂的数据Schema；
  - 在全球化部署的同时保证强一致性；
- 对比MegaStore，Spanner提供更为高级的支持有：
  - 在支持同步复制的基础之上，提供更高的读写吞吐；
此外，Spanner还能为数据提供多版本的读写，每个版本通过一个TimeStamp来标识。旧版本的数据可以通过配置化的GC来回收，释放存储空间。Spanner提供的是类SQL接口，并且支持ACID的事务操作。
Spanner作为一个全球部署的数据库，其实考虑了边缘计算的概念，比如：用户可以访问最近的副本来读取数据（数据是强一致的，所以总是能读到最新的数据）；并且能够支持配置每一个DC存储的数据，数据距离用户的距离（提升读速率），多副本之间的距离（提升写速率），以及维护多少个数据副本。
Spanner作为一个分布式的数据库，提供了两个非常宝贵的功能：
- 支持 Externally Consistency (我认为可以理解为 Linearizable)；
- 支持全球的用户同一个时刻读到一致的数据。
对于如此高级别的一致性提供保障最重要的一个功能是 TimeAPI，Google通过原子钟和卫星授时服务保证每个节点都能获得一个 <10ms 的时间戳，这个时间区间成为 Clock Uncertainty ，服务节点在CU内是需要阻塞请求的。
Spanner 架构简介
- Zone：Spanner 是通过Zones来进行管理的，每一个Zone是最小的管理单元，也是数据复制的范围（Replicas只能分布在同一个Zone中）
- ZoneMaster：主要负责与Server交互，每一个Zone会有一个ZoneMaster主管数据的分配以及副本管理。
- Location Proxy：主要负责与Client交互，告知Client SpannerServer的信息。
- UniverseMaster 我理解主要是一个控制台和可视化界面。
- Placement Driver 主要负责数据的迁移和心跳检测。
ZoneMaster + Location Proxy  + Placement Driver 几乎等于GFS中Master Server的功能。

Storage Arch
首先每一个Replica中存储的数据是形如：
(key:string, timestamp:string) -> value:string

（存储层）支持的数据结构比较单一，只有String类型；每一个Replica中都存放了成百上千个tablet，每一个tablet中又存放了大量的KV键值对。Tablet是在分布式文件系统Colossus持久化的，通过一种类似于B树的文件形式存储，使用WAL来保证故障恢复。
（业务层）每一个Replica都是运行在一个Paxos Server上的，保存相同数据的Replica构成一个Replicas Group，Replica Group会选择一个Leader对外提供一致性的读写服务。
对于Leader的任期，在Spanner的版本中会给Leader发一个默认10s 的Lease，在这10s内都会以持有Lease的服务节点为Leader，用于防止Split Brain。
Leader上会额外存储两个数据结构：
- Lock Table：用来表示Key Range的上锁状态。2PL：因为Spanner和Bigtable一样都需要满足对于长事务的支持，长事务如果使用MVCC这样的乐观并发控制性能会比较差，因为需要解决MultiVersion的冲突，导致事务不断地 Abort - Retry。所以基于悲观控制的方法来做长事务是一个Trade-Off的选择。
- Transaction Manager：用于协调分布式事务。2PC：直接充当了2PC中的Coordinator，用来发布Prepare 和 Commit 的命令。

Replication
Spanner 数据模型的概念比较 多 / 复杂，需要仔细分析。
我理解在Spanner 中， Directory 相当于是分布式系统中的 Partition，也等于是数据最小的移动单位。一个Directory 是一系列拥有共同前缀的 Key组成的。
Spanner 一个Paxos Group管理一个Tablet，一个Tablet是许多Directory的组合（为了利用空间局部性原理，将经常被同时访问的Key 在物理上就近存储）。
从应用层的视角来看，Dir是数据存放的最小单位，APP在写入数据的时候可以通过 Tag的方式指定Dir，这样就能将数据写入到不同的Region中。
在Dir内部，还有一个更细粒度的数据存储单元：Fragment，当Dir数据过大的时候，会被拆分为不同的Fragment，不同的Fragment由不同的Paxos Group存储。

感觉这块内容描述得过于细致了，理解起来有点困难。。

Data Model
Spanner 暴露的接口为：半关系型的数据模型，类SQL接口，事务。Spanner对于事务和半关系型数据模型的支持很大程度是受MegaStore启发的，因为在谷歌内部，APP通常在MegaStore 和 BigTable选型的时候，会因为MegaStore对于跨机房同步的支持是同步，强一致的，而选择MegaStore；即使MegaStore的性能会因为一致性而受到削弱。
在对于事务的支持上：Spanner 的作者认为，即使2PC带来的性能损失比较大，但是不能总是让业务在无事务的存储层上编码，建议是业务层应该通过一些方式规避2PC带来的性能瓶颈。BigTable就是因为对cross-row的事务支持不够完善而饱受诟病，Google 内部为了解决BigTable对于事务的缺失，专门开发了一套叫做 Perlocate 的系统。（据笔者了解，PingCAP很多产品 (TiDB / TiKV) 仍然使用Perlocate的方案来支持事务。）
Spanner暴露给用户的的存储结构是类似传统关系型数据库的，row, columns, SQL-Like Query 等等，但底层的存储却是一种KV的模型：通过每一个Row的PrimaryKey映射到其他Columns。
$$(PrimaryKey -> (V1, V2, V3 ...))$$
并且PrimaryKey是按照某种规则组成的自增唯一Key （索引）。
以下为一个用户存储的照片元信息对应的例子，从图中可以观察到 Spanner的接口语言，存储结构。

TrueTime
时间是一个很特殊的物理量，我们通常看到的时间只是时间的观察值，而不是时间的绝对值。

三个API
TT.now() -> int[]: TTInterval: [earliest, latest]
TT.after(t) -> bool: true if t has definitely passed
TT.before(t) _> bool: true if t has definitely not arrived

这里需要强调一下的是，在TrueTime这里的授时不是一个确定的时间（与我们通常从电脑/手机/手表上看到的时间不同）。TT返回的是一个不确定的时间范围。
对于一个写事务 w 来说，它提交的时间 无法复制加载中的内容满足：无法复制加载中的内容。

时间的来源
TT 的时间来源于两个方面，GPS和原子钟
- GPS：主要受限于信号传递有可能会失败的影响；
- 原子钟：原子震荡可能产生误差，使其与真实的时间相差较大。
TrueTime 架构
无法复制加载中的内容
整体的TrueTime 架构如图所示，每个DataCenter中包含一个TimeMaster 集群用于给Spanner Server提供时间。每个Spanner Server会启动一个Daemon 进程，每隔30s向TimerMaster发起时间 Poll，用于更新本地最新的时间。TimerMaster内部会通过一些补偿机制来调整自己的时间，使得各个机器的时间误差尽可能小，在此不赘述了，感兴趣的话可以参考Google Spanner 白皮书。
并发控制 // Concurrency Control
Spanner为了支持分布式事务，对于不同的事务类型：Read-Write / Read-Only 采用了不同的策略；为了实现强一致，在事务隔离级别的基础之上加入了绝对时间，将事务执行的效率大大提升。
不得不引出一段经典名言： 
We believe it is better to have application programers deal with performance problems due to overuseof transactions as bottleneck arise, rather than always coding around the lack of transactions.

Read Write Transaction
- 2PL 
- 2PC
- Timestamp: Commit Time
Read Only Transaction
- Snapshot + TrueTime
- Timestamp: Start Time
总结：分布式系统时钟授时方案
分布式系统中，主要有两种授时的方式：
- 通过网络授时，不同节点之间可能会存在网络延迟；
- 通过石英震荡产生时间，但石英震荡会产生较大的误差；
所以想要通过物理时钟来构建一个全序（以时间为比较单位）的分布式节点集合是不够的。基于此，谷歌为业界提供了几种解决思路:
Spanner: True Time方案，通过给每一个分布式节点配备一个GPS时间校准和原子钟硬件，来对接国际标准时间，并且通过卫星通信的方式进行授时，以及通过集群中的原子钟来提供双保险。
集中授时服务TSO：通过集群内一个发号器获取自增长的逻辑时间，为了避免单点故障，这个发号器通常使用Paxos来构成一个Group互相备份。(TiDB 使用的方案)
混合逻辑时钟HLC：可以保证同一个进程内部事件的时钟顺序，但是解决不了系统外事件发生的逻辑前后顺序与物理时间前后顺序的一致性，因此做不到: Linearizability and external consistency.



Reference
- Google F1
- MegaStore
