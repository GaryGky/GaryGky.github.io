---
title: troubles_in_distributed
date: 2022-11-15 21:46:01
index_img: /img/cover/trouble_in_ds.png
banner_img: /img/bg/tasmania.jpeg
tags: [Distributed System, DDIA]
---

## Faults & Partial Failures

对于单机系统来说，正常情况下，相同的程序执行结果都是一致的，一旦CPU执行失败就会以Crash 的方式返回错误；(**Deterministic**)

而在分布式系统中，如果一个操作需要涉及多个节点，可能会出现部分节点正常而部分节点异常的情况。由于网络延迟的存在，Client 有时候并不知道在某个节点上的操作是否成功或者失败了。(**Non-Deterministic**)

## Cloud Computing & SuperComputing

构建大规模系统的两种方式：

- 超算：HPC，一台超算能够同时配备上千块CPU，用于执行一些**计算密集型**的任务：如：深度学习中的矩阵计算或者是一些模拟任务。（Shared Everything 架构）

- 云计算：云计算是通过公网IP和以太网进行通信的，它具备动态扩容和极致高可用的特点。(Shared Disk 架构）

这两种系统的区别在于：

- 互联网服务基本都是在线实时的服务，主要是为降低延迟，提升用户体验而设计的。而超算则是服务于一些离线的服务，这对于用户的响应要求不是很高。

- 超算通常都是使用专门的网络拓扑进行通信，并且每台单机的性能非常高，保证它节点直接通信的可靠和高效；而云服务基本都是建立在普通节点上的，这些服务节点的宕机率远远高于HPC的节点，同时云服务通过以太网进行通信，这样的网络链路也是不可靠的。

- 在一个由成千上万个节点的组成的云服务中，可以认为在每一个时刻都有failover发生，这对系统的 fault-tolerance 提出了更高的要求（Google File System）。一旦系统的恢复时间拉得很长，那么有可能导致服务的雪崩。

- HPC的计算节点都是聚集在一起的；而云服务的节点为了得到更低的延迟，通常会部署在靠近用户的边缘节点上，这样系统中每个节点的通信开销将大大增加。

分布式系统的目标：基于不可靠的节点建立可靠的服务。

在传统的CS 系统中，有许多这样的例子：

- 在操作系统中，CRC或者其他校验算法能够检测出 bit级别的错误 ( checksum )。

- 在计算机网络中，建立在不可靠的IP层之上的TCP 协议能够对应用层提供可靠的通信 (TCP Reliable Commucation)。

## Unreliable Network

Shared-Nothing 架构（Spanner），每一台节点都是平等地，独立地拥有自己的CPU，内存和磁盘，节点中的通信只能通过 网络请求 (**RPC**) 完成。Shared-Nothing 架构过去成为构建分布式系统的主流方法，主要是因为这个方案比较**经济**，可以利用普通的节点来实现高可用的分布式服务。

在不可靠的网络下，如果Client 没有收到请求，是无法辨别出错的情况的，以下任一种情况都是有可能的：

- 请求没有到达 Server

- Server Failover

- Server 已经执行了Request 但是Response丢失。

处理响应不可达情况的一种常用的方式就是 **Timeout，**只要超过一定时间 Server 没有给出响应，那么就会认为这次请求失败。

### Detecting Faults

许多系统都需要网络故障探测：

- Load Balancing: 检测集群中机器的活性，如果一个Node failover了，就不应该再将请求路由到该Node上。

- 在Single-Leader的服务中，如果Leader Failover了，需要从剩余的节点中选出新的Leader。

但是由于网络延迟总是存在的，系统并不能完全掌握节点的存活情况，在一些系统中，会对failover的情况给Client 一些 提醒：

1. 如果能够检测出某个节点存活，但是TCP端口上没有绑定进程，操作系统会自动给发往那个进程的请求返回一个 FIN 或者 RST 报文。

1. 如果一个进程 Crash了，但是操作系统仍在运行，可以用一个脚本主动将进程Crash的消息Push到其他节点上。（**HBase**）

1. 网络管理员能够通过DataCenter提供的管理接口，监控DC 中节点的存活情况；

1. 如果IP层不可达，可以通过ICMP 报文的返回值判断是否有不可到达的package，进而判断Server 进程的存活情况。

### Timeouts and Unbounded Delays

如果timeout 是解决网络故障的唯一方法，工程师也不能完全确定 timeout的时长：

- 如果timeout 时间设置得过长，那么Client需要一直阻塞直到该节点declare failover，在等待的这段时间内服务不可用；

- 如果timeout时间设置得过短，(timeout之后立即认为节点failover) ，这样就会出现节点本身是健康的，而因为网络延迟 / Spike等原因触发了timeout的机制，所以被动地被认为failover。这样会造成的后果就是在网络已经非常**overloaded**的情况下，需要进行请求的重新路由，进而引发更多的机器崩溃。

理想情况下，一次网络的RTT上界 + 一次请求处理时间的上界就应该是timeout的合理阈值，但是大多数异步的网络（分布式系统），网络的delay 都是 **unbounded**的，即：没有明确的时间范围。

在公有云和共享数据中心中，实例通常是用容器来部署的，此时不同的服务可能共享的是一台机器上的资源（CPU / 内存 / Disk），如果 A Noisy Neighbourhood 占用了过多的资源，就很有可能导致当前服务的网络delay增加。

## Unreliable Clock

分布式系统的时钟是一个非常tricky的设计，每台计算机中都会包含一个**石英震荡**的元件，基于此来产生时间，但是每台机器的石英中并不是完全精确的，因此有些机器可能会震荡得快，另一些机器可能会震荡得慢，导致Cluster中的机器Local Time不一致。

现代系统除了使用石英钟之外，还会使用 NTP来对系统的时钟进行调整，即通过**网络授时**来对节点当前的时间进行统一。

##### Time-of-Day Clocks

Time-Of-Day Clocks 返回的时间就是Wall Clock Time，与 (midnight, 1, Jan, 1970 UTC) 对齐的时间。

Linux: ``clock_gettime(CLOCK_REALTIME)``

Java: ``System.currentTimeMillis()``

Linux 和 Java的这两种时间调用 call 的都是 Time-Of-Day Clocks，Wall Clock通常都是由网络授时服务产生的，所以理想情况下是一种同步时钟。但是由于不同节点到NTP节点的距离不同，所以肯定会存在网络延迟导致的误差。

##### Monotonic Clocks

Monotonic Clocks 通常用来丈量 Duration，比如：timeout 或者 service response time。Linux中的 ``clock_gettime(CLOCK_MONOTONIC)`` 以及 Java 中的 ``System.nanoTime()`` 获取的都是 Monotonic Time。

- 当一个节点距离NTP服务较远的时候，它的 TimeofDay Clock是有可能被往前拨。

- Monotonic Clock的特点是它永远是向前走的，而不会被回拨。

在分布式系统中，可以用Monotonic Clock来准确的计算 Elapsed Time (e.g Timeout) ，但是用Monotonic time 的来衡量时间绝对值是没有意义的。

### Clock Synchronization

Wall-Clock Time 需要通过网络授时或者本地原子钟进行校对，但是由于时钟的偏移，会产生较大的误差，Google 曾经公布过对服务器墙钟时间的调查结果：每30秒同步一次的时钟会产生6ms的偏移，每天同步一次的时钟会产生17s的偏移。

到此，讨论了分布式系统不同于单机系统的点：没有共享内存以及只能通过network上的事件进行通信，由于时钟不确定性和操作系统中断的随机性，会出现Unbouded的Delay，导致Client 无法明确的感知Server端出现的问题，进而进入选择的窘境。

## Knowledge, Truth and Lies

### The Truth is defined by Majority of Nodes

分布式系统的原则就是避免单点问题的出现，不能依赖一个 Node 的信息做出决定，常用的方法是使用系统中大多数节点的投票（**QUORUM**）来决定。

#### The Leader and the Lcok

ONLY One的场景：

- Leader base的分布式系统中只能存在一个Leader，否则就会出现脑裂；

- 同一把锁在任一时刻只能有一个lock holder；

- 数据库层面的 ONLY Once，一个用户只能有一个用户名（例如：工作邮箱）

在一些情况下，即使某个节点认为它是leader，实际情况却不是这样，比如一个处于网络分区的leader实际上不是整个集群的leader，或者一个持有分布式锁的节点持有锁后，因为 Long GC而过期了，导致它仍然认为自己持有锁，而继续进行操作。

如果其他的节点相信了错误的 leader / lock holder ，整个系统就会产生错误的行为。例如，在分布式锁的场景下，Client 1获取到锁之后进行了一段很长时间的GC，当Client 1恢复之后，继续向DB发起写请求，此时便会造成数据的不一致。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=OWJlNWQ0MWNmOWY3OTJmYzUzZDIwYWVkODE3NTIxNmRfU1lIblRtYWFubkxaZnZheFBRRnlseU1ka0c2a3hKWkVfVG9rZW46Ym94Y25LV3ZyeW5UUnRnM0hIVzI4MGhObHhoXzE2Njg1MTk4NDA6MTY2ODUyMzQ0MF9WNA)

上图这种情况是实际发生过的，HBase就曾经因为这种问题发生过事故。

#### Fencing Token

Fencing Token 是用来防止上述问题发生的，一个典型的 fencing token generator就是ZooKeeper 的 zid，因为它能够保证**自增**。除此之外，下游系统也需要配合fencing token完成任务，下游系统（DB 或者 其他微服务）要能够鉴别 fencing token，拒绝掉过期的 token。对于不支持fencing token校验的系统来说，需要想出一种校验方式，例如 文件系统可以将 fencing token 编码到文件名中。

Server 端校验 Client 的token 是非常有必要的，因为server 不能假设Client总是正常工作的，所以对Client 进行校验能够对服务产生一定的保护作用。

### Byzantine Fault

**拜占庭将军问题**

拜占庭将军问题在分布式系统中指的是，一个叛军节点向集群中其他节点传递错误的信息，导致整个系统无法达成共识或者达成错误的共识。在 Byzantine Fault环境下达成共识的方法通常被称为  Byzantine General problem

- 在太空环境下，计算机的内存或者CPU寄存器可能受到宇宙辐射的干扰产生一些随机的行为，但太空飞船一旦发生错误将产生十分严重的后果，所以飞船的控制系统通常都是拜占庭容错的。

- 在一个Multiple Participant 的环境中，可能会存在一些Participant 故意传递一些错误的信息。因此，直接相信另一个节点传过来的信息是不准确的。例如在比特币交易系统中（或者其他区块链系统），它们的共识是由peer-to-peer产生的，而不需要中心节点达成共识，这种情况下出现拜占庭问题的概率会更大。

Web Application的现实场景：在大多数情况下，服务端基本不需要考虑拜占庭将军问题，因为公司的机器通常都是部署在数据中心当中，很少受到辐射的干扰，而且有专业的运维同学负责监控，叛军出现的概率很小。

### Weak Form of Lying

即使在非拜占庭的系统中，也会存在一些bug 导致服务节点在集群中通信的时候传达错误的信息，比如：由于硬件问题导致的非法消息，软件的bug，以及错误的配置。一些典型的例子包括：

- 网络包有时候因为硬件或者操作系统的出错而产生错误，TCP和UDP 能过滤掉大多数被污染的包，但也有少部分被污染的包能够[通过以太网CRC校验加TCP的checksum](https://www.evanjones.ca/tcp-and-ethernet-checksums-fail.html)。一个简单有效的方法就是在应用层加上校验来进一步防止corrupt package的出现。

- DoS攻击，攻击方通过在请求中加入超大的string导致服务器内存发生OOM。

- NTP (Network Time Protocol) 在同步的时候会与集群中大多数的节点进行通信，校验本地的误差级别，只要集群中有大多数节点能够工作，就能够将出错的 NTP Server检测出来。

## System Model And Reality

**Safety and Liveness**

Safety is defined as nothing bad happening and liveness as something good eventually happens.

**System Models**

**Synchronize Model**

这个model假设 网络延迟、进程终止、时钟偏移总是有边界的，并且 RPC的调用方能够感知到以上这些延迟，基于正确的时钟做出决策。 **Partial Synchronize Model**

半同步系统在大多数情况下都表现的和同步系统一致，但是偶尔会出现超过 bound 的网络延迟、进程终止和时钟偏移。

这种模型符合现实中大多数系统，大多数情况下，网络和进程都能够正常工作，但一旦发生了延迟，则延迟的上界可以是无限大的。 **Asynchronized Model**

在这个模型中，系统不会对环境做出任何假设，unbounded network delay, process pause  随时都会发生。因此建立在异步模型上的分布式共识算法往往更加复杂和严格。

## Summary

分布式系统中常见的三种问题：

- Network Delay

- Clock Drift (NTP server and local quartz may suffer problems)

- Process Pauce (e.g Due to GC, the process has to pause for large amount of time)

[Limping Lock](https://ucare.cs.uchicago.edu/pdf/socc13-limplock.pdf)：这篇文章中讲述了如何处理**残废节点**，一个节点没有完全宕机，但是性能急剧下降（Throughput 1GB -> 1KB），这种节点往往比 dead node 更加难以处理。

分布式系统和超算系统的不同：超算系统假设系统中节点都是可靠的，如果一个节点宕机，则整个任务需要重新运行。分布式系统可以永远运行下去，如果在没有其他因素对服务造成干预的话。