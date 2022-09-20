---
layout: post
title: TSDB part(2) - WAL && Checkpoint
subtitle:
categories: Prometheus
tags: []
banner: "/assets/images/banners/home.jpeg"

---

# WAL 简介

wal是prometheus本地序列数据库的事务日志， 在发生 插入、更新和删除等操作的时候会先写入wal事务日志文件中， 然后再操作数据库

当prometheus宕机的时候可以根据这个文件进行恢复之前的数据。

事务日志这一特性也被广泛运用于关系型数据库中。 



在Prometheus的上下文中， wal文件用于存储记录事件和内存映射，这个文件主要用于在重启的时候重放所有的事务进内存中。		



# WAL文件记录的数据类型

每个请求主要包含两组数据， 时间序列的labelset的值和sample(t,v)的值。

所以每个插入和更新请求， wal文件需要记录的有两个数据结构， Series(lk,lv)和Sample(t,v)

 labelset(lk,lv)的值用于确认唯一的时间序列。这里假定Prometheus中保存所有的时间序列的地方就叫做Series， 以哈希结构进行保存，在Series创建一个新的时间序列的时候会创建一个唯一的引用映射。

保存一个sample的记录的时候， 这个sample记录就需要包含这个Series中所对应的时间序列的引用。

除了上面的这两种数据结构， 还有一种就是Tombstones， wal文件中的这数据结构用于记录删除请求中， 删除的时间序列和此被删除的时间序列的时间范围。

每个wal文件128m.

```
data
└── wal
    ├── 000000
    ├── 000001
    └── 000002
```





总结：

wal文件中记录的一共有三种数据结构， Series(lk, lv), Sample(t,v)和Tombstones。

series是一个哈希表用于记录确认唯一的时间序列

Sample是Series哈希表中一个时间序列的一组时间向量样本数据。

Tombstones记录的是删除请求的时间序列和删除的时间范围。





# wal文件截断

在上一个章节有说到， 在Head Block做持久化的时候， 会截断已经持久化的Head block的wal数据。

由于写请求是随机的，这里的随机指的是prometheus写wal文件的逻辑是随意写到其中的一个文件中。

为什么呢？可以假设prometheus写wal文件的逻辑，主要有两种

1. 并发写请求的时候多个线程顺序写一个wal文件， 当这个wal文件写满了以后再写下一个wal文件。
2. 并发写请求的时候每个线程占用一个wal文件进行并发写操作

显然第一种对于高并发系统来说吞吐率低，第二种更符合逻辑， 也符合写请求是随机的这种说法。

继续下文

由于写请求在wal文件中是随机的，所以要截断旧的wal事务日志的时候如果要正确阶段这些日志的话，那就需要遍历所有的wal文件中的事务日志， 这个过程就会很耗时，于是prometheus走了另外的一条路子， 直接截断wal文件的2/3

```
data
└── wal
    ├── 000000
    ├── 000001
    ├── 000002
    ├── 000003
    ├── 000004
    └── 000005
```

于是这上面的000000, 000001, 000002, 000003就会被截断， 但是截断的这2/3的wal文件中可能包含了还在Head Block中的Series时间序列数据和Samples数据， 怎么避免这些数据的误删呢， 这个时候"checkpoint"这个概念就出来了。

# Checkpointing

​	在进行截断WAL文件之前，从被删除的2/3的wal文件中创建checkpointing。假设截断在时间T之前的数据操作，也就是说T之前的数据都已经持久化成block了， 以上面的为例， checkpointing将会基于chunk文件 000000, 000001, 000002, 000003进行如下操作：

- 删除所有在时间T之前的时间序列Series记录
- 删除所有在时间T之前的采样Samples数据
- 删除所有在时间T之前的删除记录timestone数据
- 保留剩下的Series， samples， timestone在wal文件中。

上面的这个删除操作也是可以重写的，因为一个records可能包含多个Series， samples， timestones

checkpointing的本质就是讲2/3的的wal文件进行重放， 进行进行上面列举的这四项操作。

做checkpointing时会创建一个文件夹，命名为checkpointing.x，x为创建checkpointing设计的wal文件的最新的那个分片文件的序号，同时删除旧的checkpointing文件。

```
data
└── wal
    ├── checkpoint.000003
    |   ├── 000000
    |   └── 000001
    ├── 000004
    └── 000005
```



当prometheus重启的时候就会先对checkpointing进行replay进内存中， 然后再将x+1和后面的wal文件replay进内存中。

 

