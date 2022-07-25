---
title: VM-FT
date: 2022-05-22 13:02:09
index_img: /img/cover/vmft.jpeg
banner_img: /img/bg/seashore.jpeg
tags: [Distributed System, Distributed Storage]
---
VM-FT
本文是由VMWare于2010 年提出的一个虚拟机容错解决方案，使用的是一个Primary-BackUp的架构。

推荐阅读：为了更好的阅读体验，请移步至飞书 [VM-FT](https://lo845xqmx7.feishu.cn/docs/doccny051ESMjndx9mYC7wpg0cc)

前言
在听取了MIT6.824的课程以及阅读了VM-FT的paper之后，笔者个人感觉要让VM级别的应用实现Fault Tolerance的难度会比DB (In-Memory / Disk) 级别的应用大不少，主要是由于VM级别的机器需要关注更底层信息的一些复制，包括中断，异常等。
本文介绍方法的大致思想是使BackUp保留一份Primary的状态，当Primary出现故障的时候BackUP能够平滑接替Primary的功能，并且是对用户是透明的。
为了实现这个目标，首先在论文Intro部分介绍了全量复制和增量复制两种方法：
- State Transfer：全量地将所有信息，包括（CPU，Disk，I/O）的变化都复制给BackUP，这种brutal force的方法简单粗暴，但是对网络资源的开销太大；试想一个数据库更新一行需要将整个数据库在网络上传输一次，这显然是不现实的。
- Replicated State Machine: 将服务器抽象为确定状态机，让Primary和BackUP接受相同的Client Request，这种方法实现复杂，但是对网络带宽的占用很小，适合长距离的传输。
架构设计
暂时无法在文档外展示此内容
Component
- VMM 是该文章复制的Node主体，他是建立在Hypervisor上的一个应用，运行在虚拟机中的操作系统是GuestOS，主机上的操作系统是HostOS。
- Primary 作为一个逻辑概念，标识一台VMM节点，是Client唯一能够感知到的机器。所有的网络输入或者其他输入（磁盘、鼠标、键盘）都会进入Primary VM
- Primary VM接收的所有输入都通过logging channel转发到BackUp进行同步，以保证两者的状态相同。Backup VM指令的结果与Primary相同，但是backup的返回结果会被hypervisor丢弃。
Deterministic Replay
对于一些不确定的指令或操作：
- 随机数生成指令
- `Time.Now`这样的获取时间戳指令
- 时钟中断指令
- 多核CPU之间的并发执行指令
对于不确定的操作，VM-FT中会添加一些额外的信息来保证BackUP对于这些操作的执行结果是与Primary一致的。VM-FT中会添加不确定的指令的执行结果，在backup中replay这些指令的结果。
一个logEntry大概需要包含以下内容：
Interupt Number // 事件发生时的指令号
Type // 事件类型：网络输入 / 其他输入
Data // 数据，对于不确定指令，此数据应该是Primary的执行结果

Bressoud & Schneider 提出使用了batch update的方式在Primary和Backup之间进行同步，他们提出将执行序列划分成若干个epoch，然后每次Primary-backup之间同步一个epoch的内容，来减少信息传递的次数。
FM Protocol
- 输出要求
当Primary failover之后，Backup要能够立即接管Primary的任务，并且保证自身的信息和Primary是强一致的。
- 输出规则
Primary 在发出同步信息之后会等待Backup返回ACK之后，才把Output返回给Client。

VM-FT不能保证`Exactly Once`语义，因为Primary在返回给Client Output之后发生了故障，backup在takeover之后也向Client发送故障，此时Client有可能会收到两个重复的数据包。
paper中说这个重复的数据包是通过TCP可靠传输来保证不重复的，但是笔者认为在此基础上，Client应该保证自身的幂等。

VM-FT和GFS的 Fault Tolerance
- VM-FT备份的是计算，它对于一致性要求的实现会更加复杂一些，主要体现在对于不确定性指令操作的处理上。
- GFS备份的是存储，所以GFS的备份策略会比VM-FT更加高效和简单。
Reference
- Paper
- MIT 6.824
- NUS CS5223 Lab-2
- Chapter 5 - Replication
