---
title: MapReduce
date: 2022-05-22 13:04:06
index_img: /img/cover/mapreduce.jpeg
banner_img: /img/bg/sunrise.jpeg
tags: [Google, Distribtued System, Distributed Computing]
---
MapReduce
Abstract: A high-level idea of Map-Reduce
- Map
用户使用Map函数计算出一系列的中间 K-V pairs
- Reduce
Reduce 函数在对所有的中间K-V Pairs进行聚合（Merge）操作
假设给我十个数组，让我找出每个数组的第五个元素，那么Map函数的作用就是找到每一个数组的第五个元素，Reduce函数就是将每个数组的第五个元素进行聚合，得到一个新的数组。

推荐阅读：为了更好的体验，请移步至飞书 [MapReduce](https://lo845xqmx7.feishu.cn/docs/doccnOSf3ldikYI6JOgdn5B6Gac)

Introduction
Google对于处理大数据做了上百种special-purpose 的优化，这些计算方法用来处理大量的原始数据，比如：文档爬虫，Web日志等；也用来计算各种类型的衍生数据，包括倒排索引，每天最经常出现的请求等等。
如果在可接受的时间内完成运算，如何处理并行计算，如何分发数据，如何处理错误…都对系统设计提出了很大的挑战。
Google的工程师在函数式编程的Map & Reduce函数中获得了启发，并且将其应用到了大数据计算中，使得大量的计算能够并行而且通过重试来处理fault。
Outline
- Sec.2: 描述一些基本的编程模型 
- Sec.3: MapReduce的基本接口实现
- Sec.4: 优化
- Sec.5: 实验
- Sec.6: MapReduce在Google中的应用
- Sec.7: 未来的发展方向
Programing Model
Example：统计文件中单词出现的数量
- Map
The map function emits each word plus an associated count of occurrences, 1 in this example.
map(string key, string value):
    // key: document name
    // value: document content
    for word in value:
        EmitIntermediate(word, "1")

- Reduce
The reduce function sums all counts emitted for a particular word.
reduce(string key, Iterator values):
    // key: a word
    // values: a list of counts
    for v in values:
        result += ParseInt(v)
    Emit(AsString(result))

在概念上用户定义的Map和Reduce函数都有相关联的类型：
map(k1,v1) ->list(k2,v2)
reduce(k2,list(v2)) ->list(v2)

Other Example
- Distributed Grep: Map函数将emit一行如果改行能够匹配用户定义的Pattern，Reduce函数将数据拷贝到输出Buffer中。
- Count URLAccess Frequency：类似单词统计，每次遇到URL，Map函数都会emit 1，Reduce函数将结果聚合得到 <URL, total_count>。
- Inverted Index (倒排索引)：Map函数emit (word, document ID), Reduce 函数对word进行聚合，得到 (word, list<Document ID>)
Implementation
Map-Reduce 整体工作流程


用户能够决定将Input File分割成多个数据片段分发到不同的机器上进行处理，使用分区函数将Map产生的中间函数分布到不同的分区上进行执行，Reduce的调用也被分布到多台机器上执行，用户可以自定义分区函数和分区数量。
任务执行过程
1. 用户程序将输入文件划分为M块，通常每块的大小为16MB-64MB，然后在Cluster上创建大量的程序副本。（注意是程序副本而不是数据副本）
2. 其中一个程序是特殊的，作为Master对其他所有的副本程序 (Worker) 发起控制请求；Master总共有M个Map任务和R个Reduce任务将一一分配给空闲的Worker。
3. Worker被分配任务后会读取输入文件对应的 split，并执行Map函数，中间结果被保留在内存缓冲区中。
4. Worker会将中间结果周期性的写入磁盘，并且按照Partition函数划分为R个区域，并且将文件在磁盘上的位置返回给Master。
5. 当Reduce Worker收到来自Master的处理请求后，会向Master发来的Location发起RPC调用，读取磁盘的数据。在收到磁盘数据之后，会先对key进行排序，如果内存不足需要使用外部排序。需要排序是因为不同的key会被match到相同的Reduce任务上。
6. Reduce Worker会遍历排序后的Key，并将这个Key和它对应的中间值传给Reduce函数，Reduce函数的输出将被追加到所属分区的输出文件。
7. 当所有的Map&Reduce任务完成后，Master唤醒用户程序，并将结果返回给用户程序。
通常在MapReduce任务结束之后，会产生R个文件（每个Reduce Worker产生一个输出文件），通常用户不需要合并这R个文件，而是将他们传给另一个MapReduce函数或者分布式应用对文件进行后处理。

Fault Tolerance
Worker Failure
- Master会向Worker定期发送心跳，如果判定Worker Failure：
  - Complete Map Task or In-Progress Map Task 会被设置为IDLE状态， 并且重新执行。Complete Map Task也要重新执行，因为Worker Failure之后，其磁盘的数据是不可访问的。
  - Complete Reduce Task不需要重新执行，因为Reduce的结果是直接写入Output File的，Output File存储在GFS上
Master Failure
- Master会定期将metadata 进行checkpoint 保存到磁盘上，recover的时候加载最新的checkpoint恢复数据。
- Master如果Failure 会将任务Abort，由Client进行重试。
Semantics in the Presence of Failure
- Map 和 Reduce都是deterministic的函数（一致性），他们的输出理论上都是确定的，不论发生任何错误。
使用原子提交 (Atomic Commit) 来保证一致性：
- Map：每个Map任务执行完成后，生成R个临时文件，并将临时文件的地址封装为Message发送给Master，如果Master判断这个Message已经收到过，则直接忽略它。
- Reduce：Reduce的原子提交是依赖操作系统的重命名命令实现的 (mv in Linux), 一个Reduce任务执行完成后，会将临时文件重命名为输出文件名。如果同一个Task在多台Reduce Worker上执行，最终只有一个Worker的文件能够被保留。
空间局部性
在MapReduce提出的年代，网络带宽会成为系统的bottleneck。所以Master会考虑从GFS中寻找距离最近的Replica以减少网络的带宽使用。如果任务失败了，Master会优先在保存有Input File (Split) 数据拷贝的机器上执行Map任务。这样当执行大规模MapReduce任务的时候，大多数的Input data都是从本地读取的。(Worker 会缓存Data)，Master作为GFS的Client，不会缓存Data[1]。
备用任务
为了防止“木桶效应”，一个Worker的慢执行，导致整个MapReduce任务执行变慢，Master会在一个任务马上完成的时候，调度备用Worker来执行剩下的处于in-progress中的任务。不论是backup Worker还是初始Worker完成了任务，Master都会认可这个任务被完成。
应用场景
- 分布式Grep
- 统计URL访问频率
- 构建倒排索引
- 分布式排序：分布式MergeSort
Related System
- Distributed Storage：GFS
- Batch Processing: Spark
- Stream Processing: Flink
- High Availability: Chubby
总结
MapReduce的成功归因于以下三点：
模型使用起来非常简单，因为开放的API接口隐藏了内部的故障处理和并行机制；
大规模的应用场景能够使用MapReduce来处理，比如：机器学习，数据挖掘，搜索推荐；
谷歌的工程师们提供了一套完整的实现，使其能够应用于大规模的机器集群上。

经验
编程模型的约束使得任务的并行和计算，故障容错变得更加容易，在MapReduce中用户只需要定义Map函数，Reduce函数和一些超参就能完成对任务的执行。
网络带宽是宝贵资源，（现在都光纤了，应该不存在这个问题了吧）。
冗余的处理能够一定程度上避免木桶效应。

局限
历史局限，当时的背景下单机的性能远远不足，所以没有考虑使用内存进行更高效的计算。
没有将资源调度和计算调度分离。在后续的Hadoop生态中，MapReduce只关注与计算而资源调度由Yarn进行管理。

Reference
[1] Google File System
[2] https://research.google.com/archive/mapreduce-osdi04.pdf
[3] 
https://tanxinyu.work/mapreduce-thesis/
[4] 正在实现的：
https://github.com/GaryGky/MIT-6.824/tree/lab1-mr/src/mr

