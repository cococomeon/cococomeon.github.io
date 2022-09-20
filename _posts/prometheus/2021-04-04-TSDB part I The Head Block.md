---
layout: post
title: TSDB part(1) - The Head Block 
subtitle:
categories: Prometheus
tags: []
banner: "/assets/images/banners/home.jpeg"

---

## 一. 内存结构

![]({{site.url}}/assets/images/2021-04-04-TSDB part I The Head Block.assets/tsdb1.svg)

上图就是一个Promtheus的TSDB的一个缩略图，可以把TSDB的存储数据结构认为是由一个个按时间线性排列的Block块组成的，上图中主要包含四个部分

1. 黄色部分就是在内存中的head block
2. 橘色WAL(write ahead log)指的是事务日志
3. 蓝色为内存通过mmap的方式映射到磁盘上的数据
4. 灰色的即为在磁盘上的序列化数据 

一个sample(t,v)到tsdb的时候，先经过wal写入事务日志，然后放进Head block内存的chunk中， 当一个chunk满了以后会使用mmap将其flush到磁盘中， 然后经过一个固定的间隔时间， 默认2h， 多个chunkhui会被合并flush到disk中，形成灰色的block 1 2 3...n，然后再经过多个2h后，已经flush到disk的block会被合并压缩成一个更大的block。

### 1.1 Series 组成

一个series由两部分组成：labelset (lk, lv) + sample (timestamp，value)

一个时间序列由labelset唯一确定, labelset只保存一份，存放在index， 索引中。 

而一个sample(t,v)则存放在chunk中，chunk是一个压缩单元， 一个时间序列有多个chunk，但某一个时间只有一个chunk可以写，  

index{labelset} ---> [chunk1, chunk2, chunk3....]



下面我们就来打开一个Series里面，存储sample的的Head block中的数据结构

### 1.2 sample的生命周期

一下的讨论的范围是一个指标样本的生命周期， 对所有的指标样本来说都是一样的。

1. Head block中的chunk写入

    ![]({{site.url}}/assets/images/2021-04-04-TSDB part I The Head Block.assets/image-20210519210456864.png)

   上图中红色部分为活跃的chunk，每个Block由多个chunk组成，一个block在某个特定的时间里面只有一个chunk可以写sample数据， 这个chunk就是Head block中的活跃chunk。在写入Head block的chunk之前会先写进wal文件中。

2. chunk写满分片

    ![]({{site.url}}/assets/images/2021-04-04-TSDB part I The Head Block.assets/image-20210521144457561.png)

   当这个chunk写满了120个sample以后，或者达到了一个chunkrange跨度(默认2h)， 一个新的chunk就会建立。上图中黄色的便是一个写满的chunk编号为1， 红色的就是新建立的chunk。
   这里假设抓取间隔为15s，则120个sample需要30分钟，一个chunk写满就需要30分钟。

3. chunk映射到虚拟内存

    ![]({{site.url}}/assets/images/2021-04-04-TSDB part I The Head Block.assets/image-20210521145041960.png)

    ![]({{site.url}}/assets/images/2021-04-04-TSDB part I The Head Block.assets/image-20210521145634862.png)

    ![]({{site.url}}/assets/images/2021-04-04-TSDB part I The Head Block.assets/image-20210521145743458.png)

   从2.9.0开始，Prometheus不再将所有的chunk保存在内存中， 而是使用mmap将已经写满的chunk映射到磁盘中， 内存中只保存磁盘的映射地址。通过mmap映射技术，内存中只需要持有这个chunk的引用映射就可以动态地将chunk导入到内存中了，mmap是操作系统提供的特性。

4. chunk持久化

    ![]({{site.url}}/assets/images/2021-04-04-TSDB part I The Head Block.assets/image-20210521150009207.png)

    ![]({{site.url}}/assets/images/2021-04-04-TSDB part I The Head Block.assets/image-20210521150035246.png)

    *上面假设抓取间隔为15s，则120个sample需要30分钟，一个chunk写满就需要30分钟，一个chunkrange(默认2h)就可以写4个chunk。*

   当Head block中的所有chunk时间跨度达到$chunkrange \times 3/2$，就会将第一个chunkrange跨度的chunk压缩成一个持久化的block，这个时候wal文件会被截断，因为这个时间跨度的已经不在内存中了， 就算宕机也不需要在现在这个wal文件中进行恢复，"checkpoint"也是在这个时候创建的。

上图中的持久化的block的index索引是什么？

这是一个存储在内存中的倒排索引， 当产生同一个chunk压缩持久化动作的时候， Head block中的旧的chunk就会被切断同时发生垃圾收集动作。



总结：

从本文中可以得知，

1. Prometheus 2.9.0开始活跃样本数在没有做持久化成block之前，并不是全部保存在内存中， 而是只保存一个120个sample的chunk在Head block中，Head block中的剩余的chunk会被映射到磁盘上，如果这个chunk在2h都没有达到120（也就是抓取间隔是1分钟的时候), 才是内存中会保存2小时的数据.
2. 只有在达到了1.5*chunkrange中以后才会做持久化操作，也就是3小时，会持久化前面的chunkrange个跨度的chunk。

活跃样本数需要的内存是：

count(series) * 120sample * 2byte = 40000 * 120 * 2 = 9m



# 内存预估







原文

[Prometheus TSDB (Part 1): The Head Block](https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/)

