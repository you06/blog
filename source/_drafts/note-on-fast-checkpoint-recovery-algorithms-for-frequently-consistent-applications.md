title: >-
  Note on 「Fast Checkpoint Recovery Algorithms for FrequentlyConsistent
  Applications」
author: you06
date: 2020-12-26 13:35:54
tags:
---
最近考虑如何在数据库里使用 checkpoint 的时候总是觉得自己的思路很辣鸡，于是去研究了一下 [Calvin](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) 里面提到的 Zig-Zag 算法。

- paper: https://www.cs.cornell.edu/~wmwhite/papers/2011-SIGMOD-Checkpoint.pdf

## 写在读之前

在分布式事务里，2pc 是为了保证提交的原子性，prewrite 阶段写入数据，commit 阶段标记数据写入成功，事务的成功以 commit 的 primary key 为原子性标记，当其他事务遇到状态未决的记录时，会去查对应的 primary key 的提交状态。这种做法是的主旨是“尽可能减少读事务的延迟”，然而我想要的是一种，通过略微增加读事务延迟，来大幅减少写时 overhead 的策略，希望这篇文章所说的 checkpoint 能带来一些思路。

## 算法框架

![Algorithmic Framework](https://user-images.githubusercontent.com/9587680/103154571-20dda400-47d3-11eb-9f22-825fac9d88fe.jpg)

算法框架是用来衡量算法优劣的模拟场景，图里的 Mutator 是往内存里写入数据的应用，Logical Logger 负责写入 replay log，Asynchronous Writer 异步将内存中的数据写到盘上。当发生故障重启时，内存中的数据会丢失，而我们可以通过 checkpoint 和 replay log 重现当时的场景，replay log 需要足够小（checkpoint 足够频繁），目的是让 replay 能够在极短的时间内完成。

## baseline

论文中提到了两个 baseline 算法，Copy-on-Update 和 Naive-Snapshot。

### Copy-on-Update

### Naive-Snapshot

Naive-Snapshot 不使用 Logical Logger，仅复制内存中的状态，然后异步写入到盘上，这种做法的优点是在处理读写请求的时候，不需要检查 checkpoint，因为没有 replay log。

