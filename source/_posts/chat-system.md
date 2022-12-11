---
title: chat_system
date: 2022-12-11 21:07:21
tags: [system design]
banner_img: /img/bg/lighthouse.jpg
index_img: /img/cover/lark.jpeg
---

# IM 系统的历史

传统的消息系统是推送消息，不进行持久化，即便是传递离线消息，当接收方收到了发送方的消息之后，也会从数据库中删除对应的消息。在传统的IM系统中，没有消息回溯的能力。

现代的消息系统中，消息是先存储后同步。消息存储库保留全量的会话消息，主要用于支持**消息漫游（消息的离线推送）**。消息同步库保存的是所有未同步到接收方的消息，如果接收方在线，则这是一个更优的同步路径，消息能够直接传递到接收方，如果接收方离线，下次登录的时候会从消息同步库中拉取未同步的消息。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NGVmMTY1NWIzOTNjOTUxMTI4M2VmNjQ2NGQ5OTdmZWZfbk1yNzEyQkhURjhlck9wdnNtZVJhSzdOdTZsMXQ3VVpfVG9rZW46Ym94Y25rTExVWDR5cFRVR09Xa3VDVTllemxjXzE2NzA3NjQwNzU6MTY3MDc2NzY3NV9WNA)

# 需求分析

Chat System 就是一个IM即时通讯软件，用户A向用户B发送消息，或者用户A向群组发送消息；用户B或者群组内的其他成员能够即时地获取到A发送的消息。

所以从High Level的角度看一个消息系统应该具备以下功能：

- 消息存储：存储所有用户发送的消息，以便用户的立即读取和历史消息的回溯；

- 消息推送 (Online)：A 发送了消息之后，要立即推送到好友B的消息列表中，并且延迟要求尽可能的低；

- 消息通知 (Offline)：A发送了消息之后，B不处于在线的状态，所以需要发送系统通知提醒B去阅读新的消息

- 多端同步：不同的设备能够读到的消息内容是一致的

暂时无法在飞书文档外展示此内容

# 整体架构

**接入层**：接入层提供客户端接入服务：包括与客户端保持短连接和长链接，对于用户在线的情况 （A和B正聊得热火朝天），此时 A B都会与服务建立长连接，WS协议提供了全双工推送的能力，由服务端将消息推送给用户。

**逻辑层**：逻辑层提供业务逻辑的封装，包括：服务发现，**消息推送**，和数据持久化，用户状态更新，好友关系维护等。

**数据存储层**：数据存储服务应该采用 RDB + NoSQL混合的方式，关系型数据库可以提供用户设置，用户Profile，好友列表等数据模型的存储，NoSQL可以用于存储消息的实体，以追求更极致的性能和更好的Scalability。

暂时无法在飞书文档外展示此内容

## 接入服务

用户A在登录之后，由LB路由到正确的 机房/集群/服务节点上，请求 ZooKeeper来发现当前可用 Chat Server，并从中选择一台Assign给User A，之后User A与这台Server建立WS协议；（此时另一个User B 可能已经连接了同一台或者另一台 Chat Server），双方进行实时地聊天。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NzM2MDBiNjhiODE5ZDQ3ODg4YzUyMjNiYWFkMDA0YzBfOFF4TWwwVlNDUGhtSVN6eTdtaUFTZlJSSW01dHVyVERfVG9rZW46Ym94Y25Mc2hmN2YwZ3RqYW5nY1ZxaklrY0JlXzE2NzA3NjQwNzU6MTY3MDc2NzY3NV9WNA)

## 数据模型

### (Naive) 数据模型

大多数IM系统的 Chat Message都是使用 K/V 的NoSQL数据库进行存储的，数据模型如下，Primary Key定义了用于消息查询的键。

暂时无法在飞书文档外展示此内容

使用NoSQL作为存储层的主要考虑是：

- NoSQL 具备较好的 scale out特性

- NoSQL 的可用性较高，延迟低

### 现代IM系统存储模型 Timeline

Timeline 是一个类似 Message Broker的模型，有一个生产端 (PUB) 和 多个订阅端 (SUB): 同一个用户的不同设备。

- 每一个消息有一个独立的SeqID，严格递增，新的会话消息通过Append的方式追加到timeline 末尾。

- 接收端会维护消息位点 cursor，可以根据cursor的值定位到 Timeline中某个消息。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=OWQxNTFmMGRkNzgxY2ExNjViMzQ1OWJhOTgxOTcyMzhfRmxPNGVzZkRwZG11Y1hGRzdNRGpWUWlPdkNma0lUU2hfVG9rZW46Ym94Y25La1BBQ3lPUGFGaHoxWTVXRE1sbTliXzE2NzA3NjQwNzU6MTY3MDc2NzY3NV9WNA)

|          | 消息同步库                   | 消息存储库                |
| -------- | ---------------------------- | ------------------------- |
| 数据模型 | Timeline 模型                | Timeline 模型             |
| 写能力   | 高并发写，峰值十万级别TPS    | 高并发写，峰值十万级别QPS |
| 读能力   | 高并发读，峰值TPS为十万      | 少量范围读                |
| 存储规模 | 保留一段时间的同步消息，TB级 | 保留全量的消息，百TB级    |

对于数据库的要求包含以下几点：

- 不要求关系型，但是需要支持队列模型，并且能够支持自增的SeqID

- 能够支持高并发的写和范围读取

- 能够保存海量数据

- 能够为数据定义生命周期

Facebook Messenger 使用 HBASE作为消息存储NoSQL，Discord 使用Cassandra作为底层的存储。阿里试用TableStore进行存储，是基于LSM引擎的分布式NoSQL，支持自增列，能够完美实现 Timeline这样的逻辑模型。

## 消息同步流

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWNmNjk0ZTljZTNhMDUxZGViYzA0YjFmZjVjYWFhZGNfYUl5a0t1MFVxV0lERTB0dmIyYnFzakxXZnBDN3VmMHNfVG9rZW46Ym94Y25mTkNXV09SWkhRSE5ES05vaTZ0SkhkXzE2NzA3NjQwNzU6MTY3MDc2NzY3NV9WNA)

消息的同步模型分为读扩散和写扩散两种方式

- 读扩散：仅写入到会话的timeline中，每个用户在登录后或者打开会话的时候去会话的timeline中拉取全量的数据。

- 写扩散：写入到会话的timeline中，并且写入到会话中每个用户的timeline中。

# 飞书IM Cloud

- 用户之间的消息传递依然是通过云端进行传输，云端需要负责消息的存储

- 每一个Receiver有一个 Inbox (用户维度)

- 服务端维护维护device维度的cursor，每次使用cursor去读取inbox中的数据

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=YmZkN2VmMWExYTA2OTU2YTUyYmFiYjdmNGRlMTA1YzlfWnh0cFNxZ3VhaDBmb3hrdTlUdHh4VmRzNzRrQVFSOG9fVG9rZW46Ym94Y25TY3BwNUNtZUFESURYMHp0SUw1a01lXzE2NzA3NjQwNzU6MTY3MDc2NzY3NV9WNA)

## 消息收取

消息的收取由inbox完成，inbox需要支持以下两种操作

- Scan 通过cursor一次性拉取所有未读消息

- Append 将消息顺序写入到 inbox 中存储消息的数据结构，为了保证inbox写入的顺序性，所有对于一个inbox的写入应该路由到一个线程上

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=MmQ0MTA1YWU2ZDY3Mjk4YWUzM2EwOWU5YTE3N2Q5MmFfcWhtUWFydXVLbG9JVlFJYkJNRkhhQjBWSjVvTzlXSkNfVG9rZW46Ym94Y243RmJibE1qWHhDenZmaTA2d09zQkVnXzE2NzA3NjQwNzU6MTY3MDc2NzY3NV9WNA)

## 消息分发

整个过程可以抽象成会话分发和成员分发两步：

- 会话分发使用 会话ID作为Partition Key保证一个会话的消息能够顺序写入到第二级的MQ中

- 用户级别的MQ以UserID为Partition Key 保证从会话MQ接收到的MQ能够顺序写入一个用户的 Inbox

注意：这里使用了两级MQ，任意一级的MQ中如果有一个Partition阻塞，就会触发Kafka的扩容，导致消息乱序，例如：消息cursor已经读到了X，但是新写入的消息会因为Rebalance被放到X之前的某个位点，此时这条消息就读不到了。

原因是在当前的设计中是先生成index，然后再Append到inbox末尾的，Append相当于NoSQL的一个set操作。

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NDUyYjJjMjFjZTg2YmNlNWYwNWEyNTE3N2JkZTNjNjNfejBwZGUxa21BMkxza3RaYTN3REFYb21yMVFUeGh1MzhfVG9rZW46Ym94Y25ZVDJRYklzbWtwclA1NEFXSHk1R2piXzE2NzA3NjQwNzU6MTY3MDc2NzY3NV9WNA)

# 总结

现代 IM的架构主要是基于 Timeline这一逻辑模型展开，Timeline模型对于存储层没有关系型的要求，但是需要实现SeqID的自增。Table型的数据库能够较好地支持这种场景的需求。

基于timeline的消息存储和推送模型，还可以应用在Feeds流、实时消息同步、直播弹幕等场景。

# Reference

- [阿里云现代IM系统](https://developer.aliyun.com/article/253242)

- [System Design Interview Volume-1 Chapter 12](https://www.amazon.sg/System-Design-Interview-insiders-guide/dp/B08CMF2CQF/ref=asc_df_B08CMF2CQF/?tag=googleshoppin-22&linkCode=df0&hvadid=404658063268&hvpos=&hvnetw=g&hvrand=11142147491480852710&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9062543&hvtargid=pla-934212337151&psc=1)