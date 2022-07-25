---
title: GFS
date: 2022-05-22 12:18:25
index_img: /img/cover/GFS.png
banner_img: /img/bg/sunset.jpeg
tags: [Distributed Storage, File System, Google]
---

Google File System
- 分布式文件系统，弱一致性模型 (Relaxed Consistency)；
- 谷歌三驾马车之一，与MapReduce和BigTable开启了新的大数据时代，引领工业界向NoSQL发展；

推荐阅读：为了更好的体验，推荐移步至[飞书文档](https://lo845xqmx7.feishu.cn/docs/doccncmo8iqkFFn424B8lZd44gh)。 [GFS FAQ](https://lo845xqmx7.feishu.cn/docs/doccnadQrN9vNCbIGJh5YtTOGxh)。

Intro
单机系统的存储容量始终是有限的，随着数据量不断增大，如果只对单台机器进行scale up也会逐渐达到上限，所以scale out to multi-Server是发展的必然。
GFS主要支持谷歌内部的软件系统，比如：YouTube视频，Google Drive等等，谷歌并不对外开放GFS的接口，只对外提供上层服务。Bigtable 和 MapReduce 都是建立在GFS之上的应用。并且Hadoop生态中的HDFS也是在GFS系统之上发展出来的一个文件系统分支。

系统的主要目标和大多数分布式存储系统一致，主要包含三个方面：
- Reliable, High Performaces
- Scalable
- Available
系统设计时考虑的几个方面：
- Normal Failure
文件系统由上百台廉价的存储机器组成，对外部大量的Client 提供服务。
- Huge File
系统需要存储TB级别的数据集，由十亿级别的KB的数据组成，所以需要重新对磁盘IO和文件块大小进行评估。
- Append rather than Overwriting
在GFS中几乎不存在随机写，并且一旦写入了文件，所有的读操作都是顺序读，系统的优化主要集中在这个点上。
Assumption
谷歌的工程师针对自家的业务场景对系统的情况规定了一些前提与假设：
- 系统中每一台Sever都是廉价的主机，所以经常会fail，需要对系统进行监控和容错。
- GFS主要用于存储大文件，相比Chubby 主要是用来存储小文件的场景，GFS存储的文件量通常为几百万的100MB的文件甚至Multi-GB 的文件大小。
- 读场景主要有两种方式：
  - 大规模的顺序读写：顺序读写上百个KB大小的内容。
  - 小规模的随机读写：几KB的在随机位置的读写。
- 写场景主要是 大 和 顺序写，以Append的方式追加到文件末尾，并且GFS规定一旦文件写入就极少会被修改。
- 系统需要执行原子的高并发写入。
- 高持续带宽比低延迟更重要。
作为一个分布式文件系统，GFS在乎的是整个系统的吞吐量，由此可能会因为Replication 造成一些延迟的开销。

Interface
不完全支持标准POSIX API接口，但提供了`create, delete, open, close, read, write`这些方法。
POSIX，Portable Operating System Inferface，可移植操作消停接口，是UNIX系统设计的一个标准。Linux和Windows部分支持该协议。

- Snapshot：可以对file 或者 目录打快照
- Record Append：允许多个Client 同时向同一个文件追加记录，并且不用加锁。
Architecture
重点来了：GFS集群是一个经典的Master-Worker架构，Master充当任务分配和元数据管理的角色，而Worker是真正存储数据的角色。
整个系统的架构如下图所示。



角色
- Chunk
数据块，每一个chunk的大小为64MB并且每一个chunk由一个uint64类型的 chunk handler 唯一标识，在创建的时候由master指定。为了保证容错，通常一个chunk会包含三个副本。
目前这种设计会造成hot spot，极端情况下，所有的Client都去访问某一个chunk，然而该chunk只有三个副本，会导致chunk所在的机器 IO次数急剧增大。
GFS的解决方案是给这些chunk更多的Replica，Author们也提到了可以让Client 向另一个 Client 请求数据。

可以看到在GFS文件系统中使用了比Linux操作系统中更大的块，能够带来如下优点：
  - 减少Client 和 Master的交互次数
  一次请求就能获取所有chunk的位置信息，并且这个信息是可以在TTL规定的时间内缓存的 。
  - 减少Client 和 ChunkServer的交互次数
  利用空间局部性和TCP的长链接，减少网络IO。
  - 减少了元数据的大小
  如果chunk size设置的比较小，那么在master上就需要管理更多的元数据信息。一旦RAM装不下，就会使用Disk存储，这样磁盘IO又成为了性能瓶颈。
每一个文件会被split成不同的chunk，这些chunk可以存放在一台机器的磁盘上，也可以存放在不同机器的磁盘上。在GFS的视角看来，我需要存储和管理的文件就是chunk而不会关心file到底有多大；而从用户的视角来看，每一个file被切分出来的chunk1-n都是逻辑连续的。

- Client
GFS Client 可以认为是依赖于 GFS lib 进行类库函数调用的线程，也是一个逻辑上的概念 [2]。
Client Side 不会缓存文件数据，因为数据太大了。
- Master
提供In-Memory的元数据管理支持，主要包含：
无法复制加载中的内容
  表示Master会通过operation log持久化到Disk上。对于chunks的位置信息，通过Heartbeats 周期性获取。
Master 在内存中存储了两个表：
Table 1:
Key
Value
Filename
An array of chunk handler (nv)

Table 2:
Key
Value
Chunk handler
List of chunk servers
Chunk version number
Primary node
Lease expiration

- Chunk Server
 


Consistency Model
Guarantees by GFS
文件名namespace的变化（文件创建）是由master在内存中进行的，整个操作通过namespace lock确保操作的原子性，master的operate log定义了全局创建操作的顺序。


这张表描述了在Append Record之后系统的状态：
- Consistent: 不论Client从哪一个Replica中读取数据，Client都能看到一致的信息。
- Defined：当一个文件数据修改后，它能够保持Consistenct的状态并且所有Client能够看到全部修改的内容。是一种级别更高的一致性。
- Consistent bu undefined: Client虽然能够看到相同的数据，但是不能及时看到其他进程的修改。
- Inconsistent：不同Client能够在不同的时间看到不同的数据。
此外，表格中也显示了GFS中两种更新操作：
- Write
Write是修改原来文件中的数据，相当于是一种overwrite的操作。
- Record Append
Record Append是一种追加操作，它包含着“原子 + AtLeastOnce” 语义，因为没有AtMostOnce的语义，所以GFS是有可能存在Duplicate Record的。GFS为了保证写入的顺序，通常会由Primary Replica决定一个Chunk的写入顺序，并且由Master向Client返回一个Offset代表Record写入的起始位置。
Master 会通过心跳的方式与Chunk Server进行定期的通信，并且使用Version Number来判断Replica是否过期；一旦Master判定某个Replica是过期的，会对过期的Replica 进行GC。

Client会对GFS的数据进行缓存，并且Client有可能缓存到旧数据，由于GFS中的数据都是以Append ONLY的方式写入的，所以Client 缓存的数据可以理解为只是部分不完整的数据，而不是错误的数据。
System InterActions
Read Process
Client 向Master请求文件所在的Replica，Master返回Chunk Handler和Location；
Client向最近的Replica发起请求，读取指定范围的数据。



Write Process
Lease & Mutation Order：Master会给每一个Mutation的操作（每一个Mutation操作只对应一个Chunk，即不能超过chunksize）指定一个Primary Replica，给它赋予一个Lease，并且由这个Primary决定每chunk的写入顺序。
在Lease的时效 (60s) 内Primary可以不用与Master通信，而自行决定文件Append的执行顺序。
Master通过心跳的方式判断Primary的liveness，并且决定是否需要延长Lease的时效。



整个写操作可以分为如下7个步骤：
1. Client向Master询问哪一个Chunk Server持有要进行写操作的Chunk的Lease
2. Master向Client返回Primary的Handler以及其他Replica节点的地址；Client收到Master的回复并进行缓存，当Primary不可达或者TTL后，会向Master按照Step1重新请求。
3. Client向所有的Replica推送数据，并且Replica会将数据写入到Buffer中。
4. 当Replica都接收到了Client的数据，Client会向Primary发起Commit请求，Primary对多个写生成执行计划（主要是决定写入顺序），然后将该执行计划先应用于本地IO操作。
5. Primary将写操作转发给其他两个Replica，使其按照Primary的执行计划进行IO操作。
6. Follower给Primary 返回写入成功或者失败的Response。
7. Primary响应Client，并且返回该过程中发送的错误，Client如果收到了error，则会Reissue这次Write操作。（📢 这里会引起问题…）。
SnapShot
这个操作由于比较常见，所以在内容中增加一下。Redis持久化和一些新型的数据库引擎 (In Memory: BitCask; Disk: InnoDB-MVCC) 都使用了SnapShot来提升并发度保证一致性。
目标：尽量减少对正在进行写操作的影响。
GFS使用标准的Copy-On-Write技术来实现快照：
- 当Master收到SnapShot的请求后，首先会revoke所有Primary Replica的Lease，保证后续的Write操作都会经过Master。
- 当Lease撤回或者过期后，Master首先会将操作日志记录到磁盘，然后通过复制源文件以及目录树的metadata来将日志记录应用到内存中的状态。
- 当Client请求进行Write操作的时候，Master发现chunk上的引用技术大于1，便会重新命令chunk Server 创建一个新的chunk，在新的chunk上进行写操作。
Master's Operation
Master主节点负责的工作有：
- 所有namespace的管理工作
- 管理整个系统中所有的chunk Replicas
  - 做出将chunk 实际存储在哪里的决定，创建新的Chunk和 Replica；
  - 协调系统的各种操作（比如：读、写、快照），用于保证chunk正确且有效的进行备份。
  - 管理chunkserver之间的负载均衡，回收没有被利用的存储空间。
Namespace Management and locking
不同于其他传统的文件系统，GFS并没有为每一个目录创建一个用于记录当前目录拥有哪些文件的数据结构，也不支持文件和目录的别名。GFS逻辑上把namespace当做一个查询表，用于将full pathname（目录或者是具体的文件）映射为metadata，并且使用前缀压缩使得这个表可以在内存中被高效的表示。
Master节点的每一个操作都必须先获得一系列的锁才能够真正的运行，当Client并发的对 `/home/user/bar` 和`/home/user/foo`进行修改的时候，会对bar和foo两个文件上写锁并对父节点上读锁。

一般来说，最底层的文件修改都需要上写锁，为了防止对文件进行并发的修改，而对于其父目录通常只需要上读锁，为了防止将目录重命名、删除和SnapShot。
并且所有的锁获取都需要按照顺序获取，以此来防止死锁。

Replicas Create & Rebalance
Replicas 会在 Create, Re-replication, Rebalance的时候被创建。
创建
当Master需要创建Replicas的时候, Master会按照以下策略选择将 Replicas 放置到哪个位置上：
- Master希望选择一个磁盘使用率低于平均值的Server。
- Master不希望所有最新创建的Replicas都集中在一台机器上，因为刚创建的Replicas可能隐含着即将进行大量的读写操作。
- Master希望尽可能地把Replicas放到同一个DC中，不同的 racks，即不同的层上。
Re-Replication
- 某些Replicas不可用导致Replicas的数量低于预先设置的数量
- 程序员手动增加了Replicas的数量
Master会按照一定的优先级进行Re-Replicatoin：
- 首先，如果ChunkA的Replicas数量少于chunksB 那么优先对chunkA进行Re-Replication；
- 如果某个 chunk阻塞了Client请求的执行，那么优先对其进行Re-Replication；
- 最近活动 (Read / Write) 的文件比最近删除的文件具有更高的优先级。
为了防止过多的clone操作占用系统的带宽，Master节点既会限制chunkserver的clone数量也会限制整个GFS集群同时进行clone的数量。
Rebalancing
Master会检查Replicas的分布，然后将Replicas移动到更合适的位置。通过这个机制，Master可以逐渐对新加入的Node进行Replicas填充，而不是瞬间加入大量的Replicas将其打爆。总的来说，重平衡是为了使磁盘利用率和Server QPS更加均匀的分布到各台机器上。
Garbage Collection
GFS使用了一种惰性删除的操作，当一个文件被应用删除时，Master节点会将此删除操作立即写入日志系统，但是Master不会立即将文件空间回收，而是将其文件名设置为一个不可见的name，并且包含删除指令的时间戳。
在Master定期对namespace扫描的过程中，如果发现一个chunk的删除时间已经超过三天，会将这个chunk的空间进行回收。在一些特殊的情况下，可能会出现orphaned chunks，chunkserver如果检测到orphaned chunk会立即将他删除。

优点：这种删除方式简单可靠
- chunk可能在一些replicas上成功，在一些Replica上失败，因此会留下一些节点没有记录的chunk数据。当master发现其没有记录某个chunk信息的时候，会告知chunkserver，chunkserver会对那个空位置进行垃圾回收。

- 删除指令不一定能够送达chunkserver，在GFS的方案中，因为会有定时的心跳，因此master 最终一定会告知chunkserver删除相关的chunk。
缺点
这种方式在资源紧张情况下，无法及时回收存储空间。不过GFS提供了其他机制来确保快速的删除操作，比如动态地加快存储回收策略，在GFS不同区域采用不同的删除策略。
High Availability
Fast Recovery
Master 以及 Chunk Server不管出于何种原因的故障，都能在秒级时间恢复故障前的状态。
Chunk Replication 
一个chunk复制到三台Replica上。
Master Replication
Shadow Master：在Master节点宕机后，Shadow Master能够对外提供只读的操作。Shadow Master与Master的通信类似MySQL PB 的Binlog操作。
Shadow Master允许略微滞后于Master，但是不会读到错误的旧数据，而是读到不完整的旧数据，Append Only保证的。

Some Questions For MySelf & Reader:
- GFS prohibits the size of a record append to exceed the chunk size. Use a sequence of events to demonstrate why a larger record append may lead to correctness issues.
- When GFS can show some duplication?
- Why can Client-B read Client-A's update even though Client-A is returned an error?

GFS FAQ
GFS Supplementary 

Reference
[1] https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/gfs-sosp2003.pdf
[2]
https://spongecaptain.cool/post/paper/googlefilesystem/
[3] https://en.wikipedia.org/wiki/Google_File_System
[4] MIT 6.824
无法复制加载中的内容
[5] Google Slides