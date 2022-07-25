---
title: Transaction
date: 2022-05-22 13:01:43
index_img: /img/cover/transaction.png
banner_img: /img/bg/seashore.jpeg
tags: [Distributed Storage, DDIA]
---
Transaction
几乎所有的关系型数据库和部分NoSQL能够提供事务支持，早在1975年IBM提出的第一个SQL系统System-R中，介绍了事务的概念。虽然在过去的40年中，事务的实现方式改变了许多，但是其基本的思想与40年前的System-R几乎是一致的 ，在MySQL，PostgreSQL, Oracle, SQL Server中 事务的实现原理几乎是与System-R相同的。

推荐阅读：为了更好的体验，请移步至飞书 [Transaction](https://lo845xqmx7.feishu.cn/docs/doccnNYVPZVwOG8KKcUirv37ELM)

新型的NoSQL数据中，大多都摒弃了事务的概念，因为在支持Partitioning 和 Replication的同时要支持事务的话，会对开发者和系统都提出很大的挑战。
- Spanner 架构的NoSQL中保留了事务的概念，并且系统是强一致（Serializable）的，按照CAP理论，它的Availablity有时候会因为网络分区而收到影响，表现为延迟较高，甚至系统不可用。
  - BigTable，Chubby，ZK，GFS
- Dynamo 架构的NoSQL 已经抛弃了事务的概念，而且系统是弱一致（最终一致性）的，它的可用性较高，但是在网络分区的场景下，不同节点之间的一致性会收到影响，可能需要在用户侧执行Merge来处理冲突。
Riak (Bitcask Engine), Cassandra, and Voldemort (VoltDB) use leaderless replication models inspired by Dynamo.
目前不支持ACID的数据库系统，有时会被称为BASE (Base Availability, Soft State, Eventually Consistency) ，这是一个相对于ACID的概念，告诉用户“我的系统不支持ACID，是BASE的”，然后具体怎么理解交给用户，反正我的系统不支持ACID。
Overview
从下图中能够看到，在分布式事务中需要讨论的几个问题：
- 事务隔离级别
- 不同隔离级别的实现
- 某个事务隔离级别下的性能分析

应用系统可能会面临的挑战
- 数据库的软件或者硬件可能会在任何时候挂机，甚至是在读写进行到一半的时候宕机；（挑战了分布式系统的可用性）
- 网络分区：分布式集群中的一些节点可能无法与另一些节点进行通信，甚至集群与外部的联系也有可能被切断。
- 不同Client Propose不同的事务，导致Race Condition，或者Client可能读到一些部分更新的数据，这类数据是无意义的。这种问题在介绍索引的时候讨论过，对于MySQL这类直接覆盖记录的写法，如果不做保障，有可能更新到一半数据库突然挂机，这时候某一条记录中可能有一部分是新数据一部分是旧数据。但是对于Riak，VoltDB这样的数据库，都是采用Append的方式写入，后台使用一个守护进程定期对数据进行合并压缩，就不会出现部分更新的问题。
对于ACID一些有趣的解读
- Atomic is not about Conccurency. Perhaps abortability would have been a better term than atomicity.
- The word consistency is terribly overloaded: (Java 中overload表示重载)
  - Replica Consistency and Eventually Consistency in async replicated system (Replication).
  - Consistent Hashing is a strategy used for rebalancing. (Partition)
  - In CAP theorum, C generally means linearizability.
  - In the context of ACID, C means "good state" in database system.
- ACID中的一致性其实是应用层的概念，而不是数据库层的概念。只有Client的应用能够定义什么是一致的什么是不一致的，而数据库只负责存储应用发过来的请求，并且只会进行相对较弱的校验（如：唯一索引校验，外键校验等）。所以ACID中，只有AID是在数据库层面保证的，而C是在应用层面保证的。
Transaction Isolation
Read Uncommit
- 脏读：读到一些基本不会存在的情况，比如：事务写了之后，但是回滚了。
- 脏写：两个事务同时写一个字段，导致partial update 的情况出现。
Read Commit
实现：通常通过行级锁实现，一个事物想要读/写一个字段的时候需要先获得这条record的锁。
问题：一个很长的事务会阻塞大量的读请求，导致读请求堆积。
优化：数据库保留数据被加锁前的value，读请求在事务对record上锁的过程中可以读到old value。
SnapShot Isolation & Repeatable Read
RC存在的问题是NonRepeatable Read 和 Read Skew
问题
场景一：可以容忍Read Skew，Alice看到自己两个account的钱总和不一致了，于是她刷新了一下界面，发现钱一致了，那么不会引起panic。

场景二：BackUp如果读到了Master的old value，并且保存了Master的value，就会导致在应用副本的时候一直读取到旧数据，特别是如果Master挂了，BackUp take over之后，错误的数据会一直被使用下去。
场景三：OLAP系统中如果出现了不可重复读，会使数据分析产生偏差。
实现
SnapShot：Naive的思想是保证数据库在任何时候都能读到Consistent的数据，这样就不会产生Read Skew；在开启事务的时候打一个快照，之后所有的操作都在这个快照上进行。
MVCC的实现：每一行数据都有一个 `create_by` 和`delete_by `字段表示创建行的记录事务和删除行记录的事务。数据库通常都会有一个后台进程会不断地对产生的版本进行merge和GC的操作。

与SnapShot被一起定义的还有可见性原则，包括：
- 在每一个事务开始的时候，数据库会标记此时所有正在执行的事务，所有正在执行事务的更新会被忽略。
- 任何被已回滚的事务提交的更新都会被忽略。
- 任何被大于当前事务ID的更新都会被忽略。
- 所有其他的更新都是立即可见的。
How is the index involved in MVCC?
理论上来说，可以让一条索引指向所有版本中的记录，当某条记录被删除之后，相应的索引项也会被删除。
在实际应用中，不同的数据库产品可能会采取不同的策略：
- PostGreSQL中，禁止对某个多版本的object进行索引更新。
- CouchDB中使用appendONLY + Copyonwrite的方式进行索引更新，由于这里数据库索引的底层使用的是B+树，所以从当前节点到根节点的page都会被copy一份。
More about Repeatable Read
Nobody knows the exact meaning of RR.
因为在System R诞生的时候，SnapShot Isolaiton还没被发明出来，而初始的数据库系统对于可重复读的定义特别模糊，导致后来的数据库产品对RR都有自己的定义。Oracle 把Snapshot 定义为可序列化级别，而MySQL将其定义为RR级别，但不管怎么定义，他们实现的功能都是能够保证一个事务内的读操作是一致的。
Dirty& Write & Lost Update
丢失更新个人理解是一种 `dirty write`的情况，一般来说，并发情况下，后一个执行的事务语句会覆盖掉前一个事务语句的更新。
可能会出现的场景：
- 对于用户充值和转账的并发情况，如果充值和转账出现了并发修改一个账户的情况，如果其中一个事务覆盖了另一个事务的执行结果，假设充值100，转账入账100元，用户的账户正常情况下应该是200元，但由于丢失更新少了100元，显然是不能接受的。
解决方案大体可以分为以下几种：
- 显式上锁（Manually）：
begin transaction;
select * from account where user_id = xxx for update;
update amount where user_id = xxx;
commit;

显示上锁的问题是要求在业务层手动编码上锁：`for update`，如果程序员忘记对事务上锁的话，可能会引起丢失更新的问题。
- 借助SnapShot (Automatically)
利用DB层提供的MVCC实现对丢失更新的自动检测，一旦事务检测到两个Write Version可能会引起冲突的时候，数据库会自动让后发的事务回滚。这样做的数据库实例有：PostgreSQL。
但是MySQL的InnoDB引擎并没有提供这种MVCC的功能，MySQL的RR级别是通过Gap-Lock来实现的。
- CAS
  - 如果account_score 等于期望的old_value，那么事务可以更新account_score的值；
  - 如果account_score 不等于期望的old_value，那么事务的更新操作是有可能失败的；
问题在于CAS的比较如果基于MVCC，那么数据库基于old-version返回给事务的结果是允许更新，但实际上new_version 的值是不等于old_value的。这样就会出现竞争情况，所以在使用CAS之前需要了解数据库在CAS条件下使用的隔离级别以及隔离级别的实现方式。
update account_score set account_score = new_int
where user_id=1 and account_score = old_int

- Conflict Resolution & Replication
Lock和CAS都是假设数据库在单机上工作的场景，但是对于分布式数据库，同一份data可能在不同的node上都有副本，并且副本的更新可能是并发或者异步的 (LeaderLess / Multi-Leader)，这时候Lock和CAS可能没有办法很好的work。
对于Multi-Partition之间如何保证Atmoic Write，在Chapter 5 - Replication中提及过一些解决冲突的方式。即：DB允许在多副本上执行的顺序是不一致的，但是会通过一些手段来保证最终一致性，比如LWW (Last Write Win)，但LWW不能防止Lost Update。Riak2.0提供了多副本之间自动解决冲突并且能够防止Lost Update的方法。
总结：
- Dirty Write & Lost Update 都是为了解决对一个Key 的冲突，不涉及多个Key的竞争问题。
Serializable
Write Skew & Phantom
Write Skew是MVCC无法解决的问题。
场景
必须保证至少有一个人在Oncall岗位上，Oncaller是可以短暂离开的，但是不论怎么样，必须保证一人oncall。
问题：A 和 B同时申请Leave，此时数据库根据当前值班人员有两人同时允许AB Leave，最终导致没人Oncall。—— Write Skew

Write Skew 可以理解为：读同一条记录，更新不同的记录。在RR级别是无法防止Skew Write的，通常来说会采用以下两种方式解决上面的Write Skew问题：
- 将隔离级别设置为Serializable，因为上面 Write Skew发生的场景是DB级别的事务同时被触发，但Serializable能够使事务串行执行，有了先后顺序之后，上面的Write Skew的问题可以得到解决。
- 如果无法使用Serializable这个隔离级别的话，需要手动上锁：
begin transaction;

select * from doctors where on_call=true and shift_id=1234 FOR UPDATE;
update doctors set on_call=false where name="Alice" and shift_id = 1234

Commit;

其他场景分析
- 预定场景：预定会议室：在预定之前先查询是否存在冲突的时间，如果不存在则插入一条记录，表示预约了该会议室。
begin transaction;

select Count(1) from bookings where room_id=1
 and end_time>='2022-05-01 12:00' and start_time<='2022-05-01 13:00';
 
insert into bookings (room_id, start_time, end_time, user_id)
values (1, '2022-05-01 12:00', '2022-05-01 13:00', 'A' )

commit;

- 消费场景：防止Double-Spending：在用户消费的时候，先check用户的account中是否有足够的coins，然后加入一条Transaction记录。但是在并发的情况下，可能会插入多条Transaction记录，这样会将用户的account扣成负数，并且多送出了一个礼物，公司赔钱。
- 注册场景：不能同一个用户名：先check数据库当前用户名是否使用过，如果没有，则将该用户名写入到数据库记录中。并发情况下，可能会出现重复的用户名；但是这个可以通过DB级别的Unique Key来保证。
Phantom
问题出现的过程：而且Insert 和 Delete 无法通过 For Update解决，因为库中根本不存在能够上锁的记录。MySQL中的解决方案是：`NextKey Lock`。
无法复制加载中的内容
Materializing Conflicting
Naive Idea是给不存在的row构造锁。
比如在会议室预定的场景中，可以新增一张表用于记录一个room的预约时间：每一行表示会议室被预约的时间(roomid, start_time, end_time)，初始化所有room所有可能的预约时间，当一个事务要预约的时候，需要先拿到对应room，对应时间段的锁（这一步操作是可以通过For Update实现的），因此可以解决Phantom问题。
缺点：
- 实现非常丑陋，需要新增一张Lock Table。
- 其次是需要将并发控制模型耦合到数据模型中，造成模型污染。
实在没有办法的话，才会选择这种实现方式，更常用的防止幻读的方式是通过：`Serializable`这个隔离级别来实现。
Serializable
所有并发事务的执行过程都和某一次顺序执行的结果一致，这种方法排除了所有事务并发的可能。
Serializable实现的方法大致分为以下三种
- Actual Serial Execution
- 2PL
- Serializable Snapshot Isolation
Actual Serial Execution
采用一种暴力回避并发冲突的方式，使用单线程一次只允许一个事务执行。虽然这是一个显而易见的方法，但是这个方法直到2007 年才被数据库专家们认可。因为在之前的30多年内，许多工程师都觉得使用单线程的方式会带来很多性能问题。
- RAM变得更加便宜，使得机器的内存得到扩张，能够容量的数据集增长，使得事务能够存放在内存中，CPU能够更快的访问。
- 数据库工程师们发现：OLTP系统的事务总是特别短，通常只包含若干个 Read / Write操作，而OLAP系统通常都是需要一系列较长的Read操作组成，所以其实只要保证不出现Partial Write的情况导致分析出错，业务方大多数情况下都还能够接受，因此OLAP是可以基于Snapshot来做事务的。
一些NoSQL如VoltDB, Redis, Datomic 都提供了对可序列化级别的事务支持，使用单核单线程的执行事务其实是一种tradeoff，一方面这样做能够减少上下文切换的开销，但是系统整体的吞吐量也因为单机而受到了限制。为了让单线程的事务得到更好的支持，需要将事务打包成一个特定的数据结构或者package传给DB层。
事务包装
购买机票场景：把用户的操作和交互封装在事务中，通过多次HTTP请求完成一个事务：（查询航班 (Read) - 查询航班票价和座位 (Read) - 选择航班和座位 (Write) - 填写旅客信息 (Write) - 付钱 (Write)），这一整个事务需要不断地通过HTTP请求与用户进行交互。
- 网络 和 DiskIO 能拖垮整个系统的响应时间。
- 用户的犹豫不决也让事务访问的对象迟迟得不到释放，使其他想要购票的旅客体验急剧下降。
所以，一种方法是将“交互”的模式转变为一次性的事务请求，即：一个HTTP只能关联一个事务。即便是这样，应用层和DB层也需要通过多次的交互来完成整个事务，如下图（上）所示。

在可序列化级别下，为了保证性能甚至不允许应用层和DB层进行过多的交互，而是将整个事务包含的command一次性交给DB作为："Store Procedure"。如上图（下）所示。
不同时代的系统对于Store Procedure有着不同的支持，早期 Store Procedure 还是存在很多问题的，比如 Metric / Log等运维组件无法接入，或者一个 BadTx阻塞了整个CPU的执行也出现过。
- RDMS系统：
  - Oracle, SQL Server, PostgreSQL 分别提供了 PL/SQL, T-SQL, pgSQL/PL 的Store Procedure的支持。这些实现都非常丑陋并且没有一个配套的生态，所以在开发中使用的也很少。
- NoSQL系统：
  - VoltDB, Datomic, Redis 分别使用Java，Java，Lua来实现对Store Procedure的支持。由于Store Procedure能够在内存中执行，所以它也不会使系统的性能过度的退化；相反，它减少了网络和磁盘IO，使得系统的执行效率更高了。
总结对于Serial Execution的使用
- 每一个事务必须是small and fast的，受到 `Single CPU Single Thread`的限制。
- 它要求数据集的大小能够存在于内存中，如果数据集太大或者事务涉及冷数据的时候，就需要与Disk进行交互，此时会拖慢整个系统。
这里可以使用一个叫做Anti-Caching的策略，将访问冷数据的事务挂起，执行另一个事务，并且在这个过程中异步地将第一个事务所需的数据加载到内存中，等加载完成后就能够加载第一个事务并执行。

- 如果对于写吞吐要求较高的场景，可能需要将数据进行Partition，每一个CPU负责若干个Partition上的事务。
- 如果事务涉及Cross-Partition，则需要额外的Coordinator和锁进行处理，又会进一步拖慢系统。所以总的来说，需要根据场景进行Trade-Off，如果能够支持Dynamo通过配置化定制不同的场景会是一个比较好的方案。
2PL
第一阶段：获取锁
第二阶段：释放锁
任何可能引起并发冲突的事务都会被串行化，通过并且读写会互斥，比Snapshot级别的读写并发效率低很多。
Predict Locking
在查询条件上进行上锁。
select * from booking where room_id=123 and start_time>'2022-05-12 12:00' and end_time < '2022-05-12 13:00''

- 一个读请求必须保证查询条件上没有写锁，一个写请求必须保证查询条件上没有读锁。
Index Range Locks
Also Known as Next-Key Locking
Predict Lock对于性能有较大的损失，因为检查锁的过程非常耗时。
NextKey Lock会对事务请求的数据估计一个范围，然后对这个范围进行上锁，（通常需要配合索引使用，如果没有索引的话可能会锁住整张表）。比如：在查找会议室的时候：
- 如果通过room_id进行查找，则会在 room_id 进行上锁；
- 如果通过时间范围进行查找，则会对 时间范围 上锁。
Snapshot Isolation (SSI)
2PL 是一种极端悲观的并发处理方式，他会将检测冲突的行为前置于事务处理；SSI则是在乐观控制的基础之上加入了冲突检测机制，来判断两个事务的执行结果是否会存在并发问题，如果存在在的话，会挑选一个事务Abort。当然乐观控制也并不是完美的，在并发事务不多的情况下，能够work得很好，因为只需要回滚非常少量的事务；但是在事务并发很多的情况下，却会存在比较多的问题，因为一旦检测到事务并发冲突，就会回滚事务；这样在数据库已经接受很大流量的情况下，会使情况进一步恶化。
Decision based on outdated premise: 在预约会议室或者Doctor Oncall的例子中，并发问题的出现是由于两个事务同时基于读 数据Data，并且基于读到的Data进行分析，并且执行会影响数据Data的操作。如果数据库能够感知事务的执行结果会影响到事务过程中对Data 的分析，并对其做最坏的预估，就能够避免 Phantom & Write Skew 的问题：
- 方法一：检测MVCC 对象的Version，如果有事务更新了当前Version的Data，就认为出现了事务并发问题；

在MVVC这个隔离级别下，事务都是基于一个满足一致性的快照进行的。在SSI算法中，如果事务在即将提交的时候发现自己的premise不成立（SnapShot被修改），会立即Abort。
  - 为什么要等到事务提交才进行 Detect  + Abort操作？
事务有可能是只读的，一个只读的事务可以允许它基于一个Consistent的快照进行；在只读的事务中，只需要快照满足一致性即可，并不会出现 Write Skew的问题。而且一个事务在Read的时候，DB并不能预测事务未来是否会有写操作，因此不能 Read到快照被修改之后就立即Abort当前事务。
- 方法二：检测事务的Write操作是否会影响事务的Read操作结果。

如果事务43在写执行Write操作的时候检测到事务42更新了它的读结果，此时会将事务回滚。Index-Range Lock 需要数据库在执行事务的过程中记录，这将带来额外的开销，但是由于其能够在事务提交或者回滚之后被删除，所以并不会对内存造成过大的占用。
SSI 相比 2PL最大的好处是 Read / Write 事务能够并发，这样不会导致 Read事务因为 Write被阻塞，能够可观地增加系统的吞吐量以及降低延迟。Read 事务能够基于Snapshot进行。
SSI 相比 Serial Execution最大的好处是不用局限于单核单线程，能够最大限度地利用好多核的机器，增加系统的吞吐量。
Downside：SSL针对读写兼备的长事务不太友好，因为长事务需要一直执行到提交阶段才会进行并发检测，如果出现了Race Condition，会将执行了很长的事务回滚，这对于系统会造成不小的运行损失。

总结
Outline
- 常见的并发问题
  - 脏读 / 脏写
  - 不可重复读
  - 丢失更新
  - Write Skew
  - 幻读
- 事务隔离级别
  - Read Commited
  - Read Repeatable
  - Serializable
    - Serial Execution
    - 2PL
    - SSI
总之，不论对于传统数据库还是新型的数据库来说，事务都是一个非常重要的Feature，因为它能够保证一系列操作的ACID特性，对于保持数据的一致性非常关键。
