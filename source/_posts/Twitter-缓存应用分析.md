---
title: Twitter 缓存应用分析
date: 2022-05-22 13:03:57
index_img: /img/cover/twitter.png
banner_img: /img/bg/seashore.jpeg
tags: [Twitter, Distributed Storage]
---
Twitter 缓存应用分析

为了更好的阅读体验，请移步至 [飞书文档](https://lo845xqmx7.feishu.cn/docs/doccn329gaovix2csddT3FXFzvb)

Abstract
- TTL 缓存过期时间的设置十分重要。
- FIFO 在workload非常大的时候工作得更好（至少和LRU性能差不多，但是理解起来会容易很多）。
1 Introduction
In Memory-Cache: 
- Redis
- MemCache
将研究者的注意力吸引到：缓存命中率，吞吐量和减小延迟。目前的评估方法是存在缺陷的，大多数是基于：Storage Caching Trace，KV Database Benchmark。但是现在的评估方法都没有将Object Size考虑进去，然而Object Size是影响缓存命中率和吞吐量的重要因子。
本文基于100个Twitter缓存集群的模拟和分析，提出了以下几个重要观点：
- In-memory 缓存并不总是服务于Read-Heavy的场景，有很多缓存也是服务于Write-Heavy的场景的。
- TTL 会影响工作集的大小所以必须将TTL考虑到缓存的设置中，有效的清除缓存应该优先与缓存淘汰策略。
- Slab-Based 的缓存比如：Memcache，会受到对象大小分布不均的影响；并且瞬时增大，或者周期性变化的workload，也会对缓存造成影响。
- 根据Zipfian（奇夫定律），访问量最大的key通常是访问量第二大的key的两倍，这在写吞吐较大的系统中会出现明显的倾斜。
- 在合理的缓存空间中，FIFO和LRU在数据量很大的情况下性能差异不明显，LRU在小数据集中的性能更好。
2 Twitter 体系中使用的In-Memory 缓存
2.1 服务架构和缓存
2011 年， Twitter开始了向微服务和容器化迁移的进程，其中缓存作为一项重要的基础架构，随着DRAM的容量扩大，也随之发展。
在Twitter中，主要有两种常用的业务缓存组件：Twemcache 和 Nightawk，前者是基于Memcache构建的缓存，具备低延迟、高吞吐的特点；后者是基于Redis构建的缓存，具有更丰富的数据结构，主备提供了高可用的服务。
2.2 Twemcache Overview
Twemcache fork了Memcache早期的版本，加入了Twitter自身的一些改进。

- Slab-Based 内存管理
传统对于缓存对象的分配策略是使用 `heap memory allocators`，比如：ptmalloc，jemalloc；这些分配策略会产生“无限大”的外部碎片，而Twitter缓存对象的大小从 若干字节到 10s KB不等，传统分配策略不适用与在小容量容器中使用。
基于Slab的分配方式如上图所示，内存会被分为若干等级的Slab，每一个Slab内部又会将内存划分为大小相同的item；Slab的等级越高，表示每一个对象占用的内存越多，每一个缓存对象会被分配到大小最合适的Slab中。Slab-Based 内存分配策略防止了外部碎片，将内存碎片局限在一个item当中。
- Slab-Based 淘汰策略
当Twemcache接收到一个新的缓存对象obj时，首先会查找对应的slab-class是否存在合适大小的块，如果存在的话会直接分配给obj，否则会创建一个更大的slab-class。当内存不足时，需要淘汰掉一些内存中的内容。
Memcache 的淘汰策略基本都是基于`item`这个粒度去做的，通常使用LRU算法对内存中的缓存对象进行淘汰。这么做在 Key Size稳定的情况下没有问题，但是如果 Key Size 随着使用不断增大，后来的大对象将没有位置存储，这种情形被称为：Slab-Calcification. 
为了解决上述问题，Twemcache使用了一种基于Slab的内存淘汰策略，包括 Slab-Random, Slab-LRU, Slab-LRC。
2.3 Cache 使用场景
Twitter使用缓存的场景可以概括为三类：存储、计算和短暂的数据。

2.3.1 Cache for Storage
最常见最常用的场景，因为后台的数据库延迟高、带宽小。
该领域一些重要的研究内容包括：提升缓存命中率，重新设计一个更密集的存储设备来适应更大的工作集，增强负载均衡，增加写吞吐。
如上图所示，该缓存集群在整个业务缓存中使用的比例虽然只有30%，但是涵盖了大多数的请求，占用了大多数的内存。
2.3.2 Cache for Computation
实时流计算使用较多
包括一些深度学习框架中也十分常见（PyTorch / TensorFlow）
跟机器学习系统相关的场景接触的可能会比较多，该场景主要是缓存一些模型计算的中间变量，包括：特征(Feature), 和预测结果(Prediction)。
2.3.3 Cache for Transient Data
用来存储一些瞬态的数据，不落库的数据。
在该场景下可能会有一些数据丢失，但是Twitter的开发人员在这种高速访问和data loss做过trade-off，保留了该场景的使用。
一些典型的例子：
- Rate Limiter: 用来记录某个用户访问请求的次数，通常用于防止DoS攻击；`(UserID, Cap)`
- Deduplication Cache：用于去重。`(Key, Cap=1)`
3 评估方法
收集分析日志
Twemcache 每一个集群都有一个`built-in` 的非阻塞日志系统：klog. klog的默认设置是每一百条请求生成一个日志，为了保证分析的覆盖性：研究者手动将采样率调整为100%，每一个集群中采样两个缓存实例进行分析。
作者从153个Cluster中的306个实例收集了 80TB的日志数据，每一个Cluster的QPS为1000。为了方便分析，作者筛选了54个流量最高的集群，他们处理了90%的QPS以及76%的内存。
3.1 缓存命中率 or Miss Ratio


Miss Ratio
图a 展示了10个缓存缺失率最高的缓存服务，8 / 10的缓存缺失率在5%以下，6 / 10的缓存缺失率在1%以下。唯一的例外是最上面的点，它的缓存缺失率高达70%，它是一个write-heavy的缓存，相比其他的CDN缓存，明显缺失率要高很多。
Miss Ratio Stability
除了缓存缺失率，缓存缺失率的稳定性也十分重要，因为对于一个后端系统来说，往往是最高的缓存缺失率决定了系统需要提供的QPS能力，缓存缺失率稳定性定义为：$$\frac{mr_{max}}{mr_{min}}$$。很多时候一个稳定性更高的缓存服务比一个不稳定的缓存更受青睐。从图b可以看到，缓存不稳定性越高的服务，它的缓存命中率更低，说明后端系统对于Corner case的考虑不足，导致系统偶尔会出现一些spike。
3.2 请求率和Hot-Keys

- 蓝线：请求量；
- 红线：访问的object数量；
大多数时候对于spike的理解是hotkey导致的，但是系统并不总是这种情况，因为从右图可以看到在request出现spike的时候，object 访问量也会上升，说明不是因为热key引起。某些可能的原因是：客户端重试，网络拥塞，scan-like请求，周期性任务。
3.3 Operation on Cache
Write Heavy Cache：写操作超过30% 的workloads。

- 从图b可以看出，超过35%的缓存集群都是write-heavy的，并且有超过20% 的缓存集群的写操作超过50%。
一些write-heavy的场景：
- 频繁更新的数据：机器学习的场景，用缓存保存一些不需要持久化的数据，所有的计算和更新都是在缓存上操作的。
- 预加载的数据：记录用户行为的场景，很多服务都是懒加载的，但在用户行为分析的场景中，根据局部性原理，可能会将用户的信息一次性加载到缓存中。但这些数据又不是可复用的（一个用户看完之后就应该要删除了），所以这些预加载的数据会被频繁更新。
