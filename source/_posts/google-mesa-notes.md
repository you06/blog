title: Google Mesa 笔记
author: you06
date: 2023-06-06 17:51:50
tags:
---
## Background

Papar: https://research.google/pubs/pub42851/

Mesa 是 Google 用于分析广告数据的 AP 数据库，既然是最重要的业务，当然有最高的要求：

- 能够存储 PB 级别的数据
- 极高的写入吞吐，百万级别
- 分钟级的可见
- 事务原子性
- 跨地域部署和容灾
- 分区一致性
- 可用性
- 百毫秒级别的查询延迟
- 在线 schema 变更
- 滚动升级

以上这些问题，没有一个现有的 data warehouse 系统能够全部解决，所以需要 Mesa。

为了支持可重复读，Mesa 的数据是多版本的，同时为了保证一致性，Mesa 建立在 Paxos 之上（看人家 Google 就不整多写多读的活）。

我们先来批判一下某些现有的 AP 系统：

- 非实时，一般在使用这种系统时，用户会选择几分钟导入一次增量数据，这能够满足一些非实时的需求，有的企业喜欢每天跑批的，用用这种系统就差不多得了。
- 吐槽了一些一些论文工作的不足，有的没有考虑到超大数据，有的只考虑了 in-mem 的 AP 问题处理。

这篇论文的工作：

- 提供了事务 ACID，Google 真的很在意这一点…
- 整了个新的 Version Control 系统，能够用 batch update 来提高吞吐（牺牲一点延迟）
- Data 异步复制，meta 同步复制，异步复制能够让吞吐更高
- 大量的 schema change 不影响性能，估计支持并发
- 能够应对数据损坏

## Storage

Mesa 在定义 table 的同时要求定义一个 aggregation function $F: V \times V \to V$来做数据聚合。这个 function 要满足结合律（方便做 batch 优化），最好满足交换律，这样子我们可以以一种更随意的方式来定序。简单的例子是 SUM 满足交换律，但是一个 register 模型不满足，他们的优化手段自然会有所不同（SUM 的性能会更好）。

当然实际上的 V 会是一个 tuple，所以需要为 tuple 的每一个 element 分别定义 aggregration function。

做 batch 时存在一个 trade-off，batch 越小，latency 越低，消耗资源越频繁，占用空间越大；反之 batch 越大，latency 越高，整体的吞吐就越高，占用空间也越小。在我们定义了 aggregation function，如何做 batch 在理论上就变得很简单了。

查询时会使用一个 version number n 来进行，为了保证可重复读，需要确保 version < n 的 update 全部已经写入并且可读。

为了保证 monotonic，Mesa 需要按照 version 的顺序 apply update，这样使得 Mesa 能够处理先加后减的 case。

Mesa 认为以行为单位存储多版本会有问题，体现在三个方面：

1. 消耗的空间太大
2. 查询的时候遍历所有太慢
3. 在查询时将它们聚合起来也很慢

所以 Mesa 的多版本实际是存的 delta 数据，因为它至少满足结合律，只要按顺序聚合运算 delta，最后就能够得到正确的结果。但是他们认为 delta 数据会比整个 V 小得多？

显而易见的，当 delta 太多时，也会带来很大的空间消耗和查询计算开销，Mesa 通过 compaction 在降低 delta 精度的同时压缩大小和查询时计算的 delta 数量。简单的，将 10 条 delta 压缩成一条后，我们就无法在这个 delta 的时间区间中做更细致的查询了。但是对于时序数据而言，稍微旧一些的数据往往不需要太高的精度。这种 compaction 策略是可以配置的。看到这里感觉所谓的时序数据库已经被干裂了。

Mesa 在存储数据时，会把数据转化为列存格式，这样在某些查询中就只需要读取部分列而非整行，减少了数据的读取量，并且尽可能用连续数据来满足查询的需求。

每个 data file 都有一个对应的 index file，index file 存了每个 key 的 offset，定位到这个 offset 之后，就可以读到 base data 并且进行 delta 的累加，因为 index file 里的 key 长度是固定的，所以它可以对给定的 key 进行二分检索，速度会很快。

## Implementation

前面基本把理论都讲的差不多了，然而实现往往比理论困难得多，Mesa 将无状态的服务拆分成 update/maintenance 和 querying 两部分，看名字就知道前者是写入路径，后者是读取路径。Mesa 将状态存储在 BigTable 和 Colossus 上，前者存 meta，后者存 data，不愧是 Google，分布式 KV 和文件系统都是开箱即用的状态。

update/maintenance server 能够扩展的关键在于，controller 能够按照 table 维度进行拆分，防止了全局的单点。而一个 controller 的能力基本上足够控制一个表上的 update 操作。

在 querying server 的实现上，Mesa 考虑了需要低延迟的小查询与超大查询共存的问题，怎么调度就没有展开说。另外 querying server 是对等的，但是 Mesa 会考虑数据亲和性，将对相同数据的查询放到同一个 querying server 上面去执行，这样可以有尽可能多的 in-mem 查询（而不需要从 Colossus 上读文件），具体做法就是秘密了。

前面我们还提了跨地域部署的需求，为了做到这点，Mesa 把上面讲的 update/maintenance 和 querying 统称为一个 instance。一个 instance 是在一个 data center 里的服务合集，存了一份完整的数据，所以在 instance 内部，处理查询是简单的，读本地文件就行了。为了管理这些 instance 的写入，Mesa 又引入了一个叫 committer 的东西，这个东西本身也是无状态的，每个 instance 都会带一个，但是它会用一个叫 versions database 的东西来管理 update batch 的 version number 分配和 version 对应 batch 的状态。这有点像 etcd 做的事，如果我们要把所有的状态都用一个 paxos state machine 管理起来，那肯定会遇到吞吐的问题，但是我们只让这个 state machine 管理 meta 信息，就能把整体的吞吐放大到一个很客观的数值。

## Enhancement

Mesa 简单说了说他们的优化，但我感觉这只是他们觉得比较有代表性的：

- delta prune，如果 delta 设计的时间完全没有落在查询区间内，可以跳过这个 delta
- scan-to-seek，在 scan 的过程中利用 seek 跳过不可能的 row 来加速查询
- resume key，如果一个 querying server 挂了，Mesa 可以用 resume key 在另一个 querying server 上继续未结束的查询，大概是经历过跑超大查询被中断的痛吧

如果一个 key 上的版本（delta）很多，即使有上面所说的 compaction，累加计算这些 delta 还是会很慢，这里 Mesa 用 MapReduce 来做并行，再连续的 delta 记录中用 sample key 的概念来做拆分，个人感觉这里不是什么难做的点，因为它的 row 是满足结合律的，嗯加并发最后按顺序加起来就行了（如果还满足交换律的话都不用考虑顺序）。

Schema Change 是一个绕不开的问题，Mesa 要求 online schema change，但是根据使用场景不同分了两种方法：昂贵的万金油 A，便宜但是能满足大部分场景的 B。

A 的方法是复制整表，然后把 update 同时写入旧表和新表，当两张表的进度一样并且 query 完全切换到新表之后，对应的 table name 将指向新表。

相比之下，B 就不需要复制整表，切换过程大概做了两件事：1. 使用新的 schema，2. 写下包含所有 schema 变化的 delta 记录。这种感觉就适用于 add column 这种简单的事情。

Mesa 觉得文件级别的 checksum 不够用（如果有多处错误，checksum 可能阴差阳错的相同），为了防止这种阴差阳错的情况，Mesa 在每次 query 和 udpate 时都会校验一些基本约束，比如 key 的顺序性。Mesa 还在 instance 之间进行多种 checksum 的检验。当发生 data corruption 的时候，Mesa 可以从其他 instance 直接复制 table，如果要复制的数据太多，还可以从备份恢复到旧版本后回放最近的 update。

## Conclusion

如果一个在 2014 年之后出生的流数据库不参考 Mesa 的工作是不行的，但是 Mesa 也多次提到了这是 Google 的需求，不一定适合所有人，有这么多数据的公司不多了。
