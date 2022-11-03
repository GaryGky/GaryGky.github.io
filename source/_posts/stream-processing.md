---
title: stream-processing
date: 2022-11-03 09:56:34
index_img: /img/bg/seashore.jpeg
banner_img: /img/cover/stream.jpg
tags: [Big Data, Distributed System]
---

# Stream Processing

[Batch Processing ](https://lo845xqmx7.feishu.cn/docx/doxcnOr5SPHoYCqX6uQZdPTxyle): The Principle behind MapReduce

流处理与批处理任务不同点：

- 批处理任务通常都是离线任务，它要求输入是有限的，比如使用MapReduce任务进行排序，则要求文件中已经完整保存了所有需要被排序的 item。而流处理的任务通常都是无限的，比如：用户的行为记录，消费记录，这些数据每时每刻都在产生。

- 如果使用批处理处理流式数据，则需要将数据按照某个维度进行划分（例如：时间），例如每天或者每小时对数据进行一次划分。这样做会使数据的反馈延迟一定的时间，不适用于一些实时的任务。如果使用流式的方案，那么每当产生一条数据记录的时候就会消费这条数据记录。

- 批处理任务的**输入输出**通常都是分布式文件系统上的文件，而流处理任务的输入通常为一条数据记录，或被称为一个事件 (**event**)。

在计算机系统中，**流数据**表示源源不断产生的数据，比如：Unix的 STDIN 和 STDOUT, Jave FileInputStream API, TCP Connection 等。

流式数据通常有一个生产方和一至多个消费方，将生产者或消费者连接起来的桥梁可以是一个文件或者一个Database，消费者通过**轮训**文件或者DB来检测是否有新的event产生。但对于大部分系统来说，轮训必然增加系统额外的开销，所以如果消费者在event产生的时候能够被通知 (**notify**)到就好了…

数据库系统通常不会提供notify的机制，传统的RDB中提供了一个trigger的方案用于在某个表被更新的 时候执行某个操作，但是功能十分有限而且不好维护。

## Message System

一个常用来 notify 消费者工具就是消息系统 (**消息队列**) 。Unix 管道和TCP连接都是使用直接的通信管道连接双端的，并且只允许在两个进程之间进行通信。消息队列扩展了其功能，使得多个生产者可以向一个topic发送消息，以及多个消费者可以从消息队列中消费消息，形成一种**发布/订阅**的模式。

为了区别不同消息系统的能力，通常会考虑以下两个维度：

- 如果生产者的生产速率大于消费者的消费速率怎么办？

为了应对生产消费速率不匹配的问题，通常可以使用三种方案：1. 限制生产者的生产速率，2. 对消息进行缓冲（Buffer），3. 直接丢弃消息。TCP和Unix管道采用Buffer的方式限制速率，在网络系统中，这个过程通常被称为流量控制。

对于采用buffer 限流的系统，一个重要的问题就是如果buffer容量超出了内存限制，Queue是否会Crash 或者 Queue中的数据是否会被持久化的磁盘

- 如果消息队列中途Crash了，消息是否会丢失。

在数据系统中为了保证数据不丢，需要对数据创建副本或者本地持久化，这样会对系统的性能造成一定的影响。批处理为开发者提供了一个良好的可靠性保证，因为一个批处理任务是原子的，要么全部成功要么全部失败，不会存在Partial Write。

#### Direct Messaging

消息系统通常都会在生产者和消费者之间建立一个直接的通信链路，不会经过中间节点：

- UDP多播在金融系统中广泛应用，例如：要求低延迟的股票推送。尽管UDP是一种不可靠的协议，但是应用层可以通过记录哪些包丢失了，进而重传这些包。Brokerless的消息队列，向ZeroMQ, nanomsg等采用了类似的方案，它们使用了TCP + IP 多播的方式进行通信。

- 如果消费者在网络上暴露了一个服务接口，那么生产者可以通过HTTP或者RPC 请求的方式将消息推送给消费者。

仅仅依靠传输层的可靠传输是不够的，在消费者Crash的情况下，消息无法被接受；生产者Crash的情况下，丢失的消息无法进行重传。一个常用的方案是通过broker节点来发送和存储消息。

## Message Broker

> Broker 才是消息系统的核心，也是狭义的消息队列。

消息队列类似一个特殊的DB系统，它本身作为一个服务的提供方，生产者和消费者作为服务的Client 连接到消息队列的服务上，消息可以被无限持久化在Broker上或者有限存储于内存中。

和传统DB相比，MQ不同之处在于：

- 对数据的**持久化**：大多数MQ对于数据的存储都是临时的，一旦消息被消费，会被立即或者延期删除；而DB对于数据记录的保存是永久的，除非用户显示对数据进行了删除。

- 因为MQ对于数据的临时保存机制，所以在设计MQ的时候，通常都会认为其工作集较小，如果broker需要存储大量的上游消息，那么就需要将消息持久化到磁盘上，因为磁盘IO的限制，所以整个系统的吞吐将受到影响。

- DB系统支持二级索引等更加丰富的查询方式，而MQ只提供了一些topic 匹配的查询方式。

### Multiple Consumer

当系统中存在多个Consumer消费同一个topic的情况时，需要考虑以下两种消费模式：

- **Load Balance**：Broker负责均匀地将消息传播给下游的Consumer

- **Fan Out**：broker将消息广播给所有的Consumer，每一个Consumer独立地对消息进行消费。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjYxNGNjMGRlMzUzMzkwYjM5Nzk1OGU1NmVlMmJhYjRfaEluZXhHUTloQUg3MzNScmpkWk1abTNrWWhnM0VkaTNfVG9rZW46Ym94Y25Ec0Q5Skt5SVhNcE94c2RCa2huRzVkXzE2Njc0NDA3OTk6MTY2NzQ0NDM5OV9WNA)

在真实的消息队列中，通常一条消息会广播给所有的消费者组，每一个消费者组内只会有一个消费者实例接收到消息。

### ACK

ACK机制是为了防止Consumer处理消息失败而设计的，消息的持久化保存是由Broker内部的持久化机制保证。如果Broker给Client发送消息之后，Client 因为Crash 没有给Broker返回ACK，此时Broker就会选择另一个实例对消息重新投递，以此来保证At Least Once语义；这里会出现ACK在网络链路上丢失的情况，所以消息有可能被重复投递，需要业务下游对消息做幂等处理。

在Load Balance的情况下，会出现消息乱序，如下图所示，Broker 给 Consumer 2 发送的 m3消息丢失后，重传给 Consumer 1，但是在此过程之间 m1 接收了消息 m4进而导致消费顺序变为 (m4, m3, m5)。乱序消费在event 独立的时候不会对系统产生影响，但是如果消息之间存在 **casual dependency**，就会产生问题。

为了保证顺序消费，一种方法是 只使用一个Consumer 实例，并且维护一个队列，按顺序处理消息。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjcwYTUxYzhhMjY3YWE2NmI0YTg0ZTg0YzYzNTUyYmRfRTV4TFdRbDY4alZnNHpMMTdkSDBudEhvOEJSeFAyTVpfVG9rZW46Ym94Y25NYlZmSXVqN1FwNk1FUDMxWGc5Y2ZiXzE2Njc0NDA3OTk6MTY2NzQ0NDM5OV9WNA)

### Partition Logs

Log 是一种Append Only的数据结构，存储系统中的LSM Tree 和 WAL 都是基于Log实现的方案。在消息系统中，生产者可以将消息以Append ONLY的方式顺序写入Log，消费者可以从Log中顺序的读写。与 Unix/Linux 系统中，``tail -f`` 的效果类似。

Apache Kafka, Amazon Kinesis Streams 都是基于Log的方案实现，这些消息系统在满足消息持久化的同时仍然能够保持百万级吞吐背后的原因是因为 Log Partition，并且Replication 也保证了系统 Fault Tolerance 的能力。值得注意的是，消息在Partition内有序，Partition之间无序；Partition内的消费进度由Consumer保存，如: Consumer Group 中维护的 ``offset for xxx``。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NzMxNmU3MmFjODkyN2Y1ZDRlMTYyZmQ0ODBhNzU0NTRfTzRVcGRES1RBT2ZKUzJrSEZybkN2WVpHMFBONzlGTm1fVG9rZW46Ym94Y25ub2pQNUtqeldSVWxDMUdrd1V0SXpmXzE2Njc0NDA3OTk6MTY2NzQ0NDM5OV9WNA)

Log-Based 消息系统不需要使用Fan-Out的模式进行通信，Broker通过将某个Partition 分配给消费者实例的方式实现Load Balancing，每一个消费者实例只需要去消费被分配到的Partition中的消息，而无需关心其他的Partition。这种做法存在两个弊端：

- 消费者实例数的上限为Partition的数量，因为一个Partition只会被分给指定的Consumer。

- 如果一个Partition内出现了Slow Message，这个Partition有可能出现消息堆积的问题。

关于MQ选型：

- 不需要消息有序 & 单条消息消费时间很长： 选择JMS/AMQP的方式。

- 消息有序 & 单条消息消费时间短：选择 Log-Based & Partition的方式。有序性保证：将消息往同一个Partition中投递。

### Consumer Offset

Consumer Offset 用来标记消费者的消费进度，Broker无需关心每条消息的ack，只需要定期check消费者的消费进度。这里的 Consumer offset 和 存储系统中的 Log Sequence Number 类似，用于Follower 与Leader之间进行State Replication，在消息队列中表现为：Broker = Leader, Consumer = Follower，Follower 需要定期获取Leader的日志来同步自身与Leader的状态。

如果消费某个Partition 的Consumer A Crash了，则系统需要将这个Partition分配给其他的Consumer B，并且从最后一个记录的offset开始消费。此时如果Consumer A消费了某个区间的消息，但是Broker还没来得及记录；Consumer B就会重复消费该区间内的消息。

### Disk Space Usage

如果不对Log进行回收，任其无限增长的话，最终将会占满所有的存储空间（内存 / 外存），所以消息系统需要定期对Log进行回收。从存储的视角看，MQ中的Log是按照Segment进行组织的，系统也会按照Segment的维度对 Log进行回收和清理。

在磁盘上，系统会维护一个环形的Buffer用于存储历史 Input Log，当Buffer达到full size的时候，将会对消息进行清理，通常这个环形的buffer能够保存1-2周的时间。

### When Consumer cannot keep up with Producer - Message Accumulation

之前讨论过如果消费速率低于生产速率的三种处理方式：**Drop**, **Buffer**, **BackPressure。**Log-Based的消息系统通常采用Buffer的方式处理此问题。如果Consumer 消费速率低于生产速率，那么消息的消费位点会离生产位点越来越远，如果没有人工干预，最终会导致该消费者无法消费到消息。因为buffer无法无限增长，达到 fullsize后，系统便会对其进行清理。因此，需要对消费进度进行监控，一旦发现消息堆积，则需要人工介入排查问题。磁盘上buffer的容量足够大，因此为RD提供了充足的时间去排查问题。

与传统MQ相比，Log Based MQ 在消息被消费后不会删除消，所以消费可以被视为一个**Read ONLY**的操作。因此可以通过Consumer获取任意时间的Offset对MQ的消息进行**重放，**这与Batch Processing 的处理过程非常类似，下游与上游的数据不耦合，所以中间的处理过程可以无限执行。

## Database and Streams

### CDC: Change Data Capture

一个大型应用底层通常需要多个数据系统来支持上层的功能，比如：OLTP数据库用于支持事务型查询，Cache用于加速访问率较高的资源，Search Index用于支持搜索，OLAP数据库用于支持分析型查询。不同数据系统之间通常使用ETL（Batch Process）进行同步。

如果全量同步数据较慢的话，通常会使用 **dual write (双写)，**由应用层显示更新两个不同的数据源。但是双写存在一些数据不一致问题：

- 两个Client 同时更新DB 和 ES的同一个Key，但是更新的顺序为 DB: (A -> B), ES: (B -> A) 这样，DB和ES的数据将会永久不一致。

- Partial Write：如果更新ES成功，但是更新DB失败，两边数据也会永久不一致。如果需要保证强一致，则需要使用 2PC，并且承受 2PC带来的巨大性能损失。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NjQ1ZWNhOTkzYWEwNzA5MTVhNmEwMTBjMGUxNzkzOThfcm1oWlBKYlJESVRmRmZqUWJWbHRidE5INm9zclNFSFBfVG9rZW46Ym94Y240c1RaZ0ZvcEVDaFByMGhpQ2FFOGdkXzE2Njc0NDA3OTk6MTY2NzQ0NDM5OV9WNA)

从图11-4来看，DB和ES是两个不同的集群，集群由多个服务节点组成，每个集群只存在一个Leader，那么上述系统将表现为一个MultiLeader的系统，为了保持数据一致，需要引入额外的 CR (Conflict Resolve) 层。

如果能够将上述系统架构成一个 Single Leader的系统，Leader产生Log，Follower消费Log与Leader进行状态同步，那么系统将变得简单许多。

以MySQL为例，可以通过Binlog监听数据库的变化，并且将数据库的变化以相同的方式应用于其他的数据源。如果能够保证不同的数据源按照相同的顺序执行数据更新的操作，即可保证不同数据源上数据的一致性。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=YzgwZTI2ZjE5ZjU5MWRjYTg3NjhiYWI0ZTRhMWQ0YTdfUW5pUXV5TVZ1QWp2WEZWYmE1c290V1RzU1JJNmU3QXZfVG9rZW46Ym94Y254aUJNUWp5UXRwOGE1NjdiNEVPVHlnXzE2Njc0NDA3OTk6MTY2NzQ0NDM5OV9WNA)

另一种方式是使用数据库触发器来实现对数据变化的监听，但是触发器的方式存在较大的性能瓶颈而且支持的功能也相对有限，所以大部分场景下，还是会通过监听 Binlog的方式进行**数据同步**。

这种方案被广泛使用在：**LinkedIn Databus, Facebook Wormhole, Yahoo! Sherpa**

注意：BinLog只是为 MySQL 实现CDC (Capture Data Change) 的一种形式，Maxwell 和 Debezium 在其上开发了Binlog与Stream结合的机制；在其他数据库中表现类似，但是有不同的名称，[Bottled Water](https://www.confluent.io/blog/bottled-water-real-time-integration-of-postgresql-and-kafka/) 给PostgreSQL 实现的一个用于解析WAL Log的API，MongoRiver 可以读取MongoDB 的**opLog**，GoldenGate为Oracle 实现了类似的机制。

#### Log Compaction

和数据库引擎类似，Kafka 也提供了对于日志压缩存储的功能，这使得日志能够持久化的存储在磁盘上，当系统接入一个新的Consumer后，可以从 index-0开始读取日志并重放，以此来获得上游数据源的最新数据。

### Event Sourcing

- CDC将DB看做是一个可变的table，每次更新操作都是通过底层的 Replication Log同步到另一个数据源，Stream的过程保证了数据变化的过程是同步的。

- Event Sourcing：Application对于DB的更新由App自身基于不可变的table产生，在Event Sourcing中，event store是追加写的，没有update 和 delete的操作。Event反映上层应用发生的事件，而不是底层数据库状态的改变。

单条event log本身不是非常有用，因为用户总是希望能够看到系统的最新状态而不是系统的修改记录，所以基于event log设计的应用需要将event log流转化为应用的state展现给用户，比如：从一系列购物车加减操作中计算出当前购物车中有什么商品。这种计算方式对于 log compaction的操作与CDC略有不同：

- CDC的操作日志中越新的日志保存的状态也是越新的，所以当获取到更新的状态之后，记录之前的状态的Log就可以被系统回收了。

- Event Sourcing 基本上需要将所有状态变化保存下来，通过历史event 构建系统当前的状态。一个可以优化的地方是使用 checkpoint (snapshot)机制，定期对系统的state打快照，之后对于最新状态的计算则能够从checkpoint位置进行恢复。

### State, Streams, Immutability

从 Batch Process的角度来看，如果程序将Input文件视为 immutable的数据，那么不论执行多少次MP任务，得到的结果都是一致的（Fault Tolerance）。**Immutable** 这一特征在CDC和Event Sourcing中也十分有用。

但通常，我们会把DB看做是应用的最新状态，状态它自然是会改变的，因为DB必须支持 write 操作，这一看起来DB就是immutable的。但注意，DB 的State也是由一系列 event创造的状态，而event是追加写入 immutable file的，所以State 和 event 的关系可以用下式表达。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=Yjc2Y2Q4NmM0NmE4MGQ5M2Y3N2E2ODJmYmNjYjdiZDZfOVdwQlV5RWpObkdCVnlBakxhRzI5MWlQd1RjRHVYWUlfVG9rZW46Ym94Y256bWdpaUpkSjNzSU5mTmI5dGxoMVdkXzE2Njc0NDA3OTk6MTY2NzQ0NDM5OV9WNA)

可以这样理解，DB只是cache了event中计算出来最新的状态信息。在交易场景中，DB可以视作一个不可变对象，每一笔交易产生的时候都会向DB中创建一条记录 (**Ledger**)，用户账上的余额是由一系列Ledger计算出来的结果。如果交易过程中出现了问题，也不会修改交易的信息，而是通过追加一条新的交易记录来弥补错账。

Cons： 需要额外的存储空间保存海量的events

Pros：

- 对金融系统的Audit (审计) 需求，监管者可能需要了解每一笔金融发生的原因和详情。

- 推荐：用户将商品A加入购物车一段时间后又删除，由于DB只保存了最新的State所以看不到购物车中有商品A，但是加购这个行为却能够说明用户对商品A 其实是喜爱的。

### Messaging and RPC

消息和RPC是分布式服务间两种通信方式，RPC通常应用于Actor 模式（请求-响应），这与基于Message 通信的不同点在于：

- RPC框架是基于分布式系统的并发和通信模式创建的，而流式处理是一种数据管理的手段

- Actor 之间的通信通常都是一次性的且一对一的模式，而流式处理可以有多个消费者消费一个事件

- Actor模式的通信可以是有环的，而流式处理通常都是pipeline的模式

## Reasoning About Time

流式处理引擎通常都需要处理事件发生的时间，比如：过去五分钟内的最大值/平均值，但是这里的时间比较是不明确的，到底是broker的local 时钟还是event发生的时间。

在批处理系统中，如果涉及到时间窗口的问题，批处理任务通常都会根据event 发生的时间进行划分，看broker本地的时间是没有意义的，因为事件发生的时间与消息队列broker的时间没有任何关系。使用event的时间戳能够保证determinitic 每次计算出的结果都是一致的。

但是也有一些流式引擎使用的是本地时间计算window，这样做的好处是简单，省去了解析event的步骤，并且在事件产生时间和事件处理时间lagging很小的时候能work，一旦出现了event 积压，导致处理事件远小于event time，这种方式还是会有问题。

### Event Time V.S Process Time

导致process time 落后于 event time的因素有很多，比如：network 延迟，broker 重启，event积压ddeng，将process time 等效为event time使用存在很多问题。在流量计算的场景中，如果下游统计服务重启之后，去消费Web Server的事件会发现在重启这段时间内出现一个流量的异动，但实际上Web Server的流量是均匀的，是 Stream Processor 内部的计算逻辑有误导致的。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=MGNjZjkwMDM5OTk4YWI2ODgwNjM0ZjM0Y2I1MjZiMWRfWDdFUXNVclVmOTFhMnRSSkpmZFNJa1o1S21Yd25pRzNfVG9rZW46Ym94Y25SUzM4cmZIdnJUMU56UEZ2VnlpRTViXzE2Njc0NDA3OTk6MTY2NzQ0NDM5OV9WNA)

### Knowing when you are ready

在决定流处理窗口的时候存在一个比较tricky的问题：如果一条消息在网络中延迟了，导致其错过了计算窗口的下沿，此时应该如何处理这条event？可以考虑两种解决方案：

- 直接丢弃该消息，因为正常情况下，延迟到达的概率很小；此时需要对被Drop的消息加一个监控，当大量消息drop时，使系统产生报警

- 生产一个correction消息

在某些情况下，生产者可以追加一条记录通知消费者“从现在开始，不会再有比t时刻更早的消息产生”。但如果不同的生产者处于不同的机器上，且每一个生产者维护其最小的时间戳阈值，则下游消费者需要保存每个生产者的时间位点。

## Stream Joins

### Stream-Stream Join

> 场景：搜索

假设我们需要计算网站上URL 最近某个时间段的 trend（微博热搜）。每当用户搜索一个关键词的产生一条event，每当用户点击一个关键词产生一条event，为了计算CTR，则需要将两条event基于``session_id`做join操作。

注意，将搜索结果存放到点击的请求中的结果和Join的意义是不一样的，因为第一种做法只能告诉系统用户在某个搜索结果中点击了某个关键词，如果用户没有做点击操作的话，则统计不到。

为了实现这种类型的Join操作，Stream Processor需要维护一个状态，比如：**过去一小时**中发生的所有events，由``session_id`建立索引。当一个搜索event 到来的时候存入对应的index 中，当一个click event 到来的时候存入对应的index中，当检测到 search index 和 click index 中有匹配的记录就可以emit一个事件表示某个搜索结果被点击。如果超过了时间窗口（此处为**一小时**），则认为该搜索结果没有被点击，也产生emit一个事件表示这个搜索结果无人问津。

### Stream-Table Join

> 场景：推荐

假设需要为抖音的用户基于它的浏览记录和好友信息进行推荐，则可以将好友信息看做是一个静态的Table，User的浏览记录看做是Stream。Stream中每一个Event包含了UserID和这个用户浏览的内容。为了实现Stream和Table的Join，一种做法是对于每一个新进的Event，将其与Table做一次Join，比如用UserID和 Friends做一次Join操作，但这类DB 的Query对性能开销比较大；另一种做法是将Table缓存在Stream Processor上，省去了远程的读DB操作。这种做法和Batch Processing中 Map-Join的思想类似，Map Join也是将一个比较小的表存入Batch Processor的缓存中，与 Input 做Join。

Stream-Table Join 与 Map Join不同的是，Map Join只需要任务执行时刻的Table 快照，而Stream Processing 需要 Table的最新信息，可以用Capture Data Change的方式获取Table的更新信息，Stream Processor 订阅Table的变化 (Binlog) 。

这样我们需要维护两个Stream，一个是User在APP上的浏览记录，一个是用户好友更新的信息。

### Table-Table Join

> 场景：朋友圈

当用户打开微信朋友圈时，如果全量的去查询用户所有好友最新的朋友圈内容，展示给当前用户的话开销太大。所以可以为每个用户创建一个朋友圈缓存，可以想象成一个双向链表，当好友发布朋友圈的时候往链表中push，于是用户的打开朋友圈进行查看就可以变成一个 Single Look Up的操作。

维护这个缓存需要包含以下四种操作：

- 当 User A 发布了一条朋友圈，则所有关注了 UserA的用户应该收到A的朋友圈 内容；

- 当A删除了朋友圈，则所有关注UserA的用户应该看不见A的朋友圈内容

- 当B 添加 A为好友，B需要能看到A所有最近发布的内容

- 当B删除好友A，B不该看见A所有的内容

其实每个用户能看到的朋友圈内容应该是基于Event Stream计算出来的某个时刻的状态（最新状态），事件基于两个Stream（Friend / UnFriend 和 Publish / UnPublish）。

另一种方式看这个场景：

```SQL
select friends.from_user_id as timeline_id, agg(posts.* order by create_time desc)
from posts Join friends on posts.creator = friends.to_user_id
Group by friends.from_user_id
```

每次朋友圈更新或者好友列表更新的时候，重新计算SQL的结果，投放到对应用户的Channel中。

## Fault Tolerance

批处理任务的Fault Tolerance 是通过 Recompute 实现的，例如一个MapReduce 任务失败了，则Worker可以重新冲HDFS中读取文件，重新执行任务；因为文件是不可变的，所以每次执行的结果都是一致的。这和Exactly Once的结果是一样的，更准确的说这是一种 Effectively Once。但流处理中的fault tolerance和批处理不一样，批处理是一次任务完成之后产生结果，但是流处理基本上不会结束，所以不可能等到它结束才产生结果。

### Microbatching and checkpointing

- Spark Microbatching

Spark Stream 采用了microbatching的机制，使用tumbling window将Stream划分为小Batch，时间窗口通常设置为1秒，如果窗口时间设置的过大，则意味着需要更长的时间才能产生结果，如果窗口时间设置得过小，则**调度 (Scheduler) 和协调 (Coordinator)器**会成为瓶颈。

- Flink checkpointing

Flink采用了打快照的方式进行容错，如果Stream Processor在某个时间Crash了，则可以加载最近一次的checkpoint并且丢弃这段时间内所有状态的更新，重新计算 state，checkpoints是由Stream中的barrier触发的，它没有固定的时间间隔。

仅依靠以上两种技术是无法实现fault tolerance的，因为它无法保证原子性：如果event的计算结果离开了Message System进入了下游系统（DB system, Downstream Service），则消息产生的影响无法被回滚。

### Atomic commit revisited

为了保证 exactly-once 的语义，我们规定：一个事件能够产生Output的充要条件是这个事件能够产生的所有影响同时执行成功，比如：Consumer Offset 增加，DB成功写入，成功发送邮件等。这些effects需要保证原子性，要么同时发生要么都不发生。在分布式系统中，需要使用两阶段提交来保证原子性，但是在Stream Processing中，可以使用比 2PC 更高效的一种方式实现原子性。

在消息系统**内部**维护其他系统状态的变化和将要发给其他系统的消息，用一个Transaction包含多个消息这种方式抵消2PC带来的瓶颈。

### Idempotence

- 所有节点按照相同顺序执行一致的 Log 

- 为了防止过期的节点重复执行 Log，需要使用 **Fencing Token**

### Rebuilding State after a failure

把State保存在本地内存中，周期性刷到持久化存储或者复制到远端的机器上

- Flink 周期性给State 打 Snapshot并且刷到HDFS上

- Kafka / Samza会把state投放到一个特殊的 topic中，用于日志压缩，和CDC原理类似

- VoltDB 通过在多个节点上执行Input Message的方式来实现fault tolerance （和 Raft类似）

## Summary

这篇主要讲述了Stream Processing的是如何服务于大型应用的，Stream 和 Batch System的原理类似，但是Stream 的实时性要求更高，并且Stream处理的数据是源源不断的。Message Broker 和 Event Logs对于Stream 来说就像Batch Processing中的文件系统。

> 两种风格的 Message Broker

- AMQP / JMS Style：broker会将一条消息发送给一个Consumer，当Consumer处理完成并且返回ACK之后，Broker会将消息删除。这有点类似异步的RPC调用， RPC调用完成后，这条消息就没有意义了（不用revert back或者读历史消息），所以可以从本地删除

- Log Based Message Broker：Broker将消息投放到对应的Partition中，每个Partition会分给一个Consumer Node，Partition内的消息有序。Consumer通过维护offset记录消费进度，同时broker会将消息持久化到磁盘上，所以消息能够被追溯

Stream Processing的几种应用场景：

- 对于event的复杂查询

- 时间窗口内做聚合（搜索过去一天最长搜索的关键字：微博热搜）

- 用于同步不同的下游数据系统 (DB -> ES / Cache / OLAP)

Stream Processing中三种 Join场景：

- Stream-Stream：统计一段时间窗口内，不同Stream的信息。（搜索和推荐场景）

- Stream-Table: 类似MapReduce中的MapJoin，本地维护Table的最新版本，与进入Processor的消息做Join操作。

- Table-Table：输入是两个Table的更新Event，将这两个Stream Join之后用于更新Table Join的 Materialized View

Fault Tolerance：

- Microbatching and checkpointing

- Transactions

- Idempotent Writes

## Reference

[1] Designing data-intensive applications: The big ideas behind reliable, scalable, and maintainable systems. Stream Processing.

[2] [Apache Kafka Documentation](https://kafka.apache.org/documentation/s)

[3] [Bottled Water: Real-time integration of PostgreSQL and Kafka](