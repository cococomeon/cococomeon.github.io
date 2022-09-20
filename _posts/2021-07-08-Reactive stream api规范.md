---
layout: post
title: Reactor系列- Reactive Stream api规范初识
subtitle:
categories: Reactor
tags: [reactive, reactor, steam]
banner: "/assets/images/banners/home.jpeg"

---


## 一. 出现的原因

java在异步编程的时候需要各种的callback回调接口， 由这些回调接口来实现异步编程，但是这些异步接口可以散落在各处，产生各种各样的异步编程问题，例如：

-  回调接口的异常、超时处理困难
- 多个异步任务协同出了
- 重构问题

为了解决这些异步编程的各种解决的办法， 其中一种就叫做"reactor programming", 就像面向对象编程，函数式编程一样， 反应式编程也是一种编程范式。

反应式编程，本质上是对数据流或某种变化所做出的的反应， 但是这个变化是未知的， 所以他是一种基于异步回调的方式在处理问题。

和传统回调的区别就是，反应式编程是先定义好一条反应链，当数据流经过这条链的时候触发这些回调， 这个是在编译前就可以知道的，而传统的回调都是在运行期才能知道回掉链张什么样子。

当越来越多人使用这种思想的时候， 自然就会需要一套规范。由此，2013年底Netflix，Pivotal和Lightbend中的工程师们，启动了[Reactive Streams](http://www.reactive-streams.org/)项目



## 二. API初览

reactor steam和jpa， jdbc一样是一种规范。reactor staeam api就是为了找到一组最小的接口， 方法和协议从而实现非阻塞式背压数据流处理。

从代码结构上看，主要包含两类接口， reactive-steams和reactive-steams-tck, tck包是Technology Compatibility Kit ， 技术兼容包

reactor steam api主要包含一下接口：

```java
//发布者
public  interface  Publisher < T > {
    public  void  subscribe（Subscriber <？ super  T >  s）;
}
//订阅者
public  interface  Subscriber < T > {
    public  void  onSubscribe（Subscription  s）;
    public  void  onNext（T  t）;
    public  void  onError（Throwable  t）;
    public  void  onComplete（）;
}
//表示Subscriber消费Publisher发布的一个消息的生命周期
public interface Subscription {
    public void request(long n);
    public void cancel();
}
//处理器，表示一个处理阶段，它既是订阅者也是发布者，并且遵守两者的契约
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```



## 三. 核心概念

### 3.1 异步和数据流。

一、数据流的产生与消费

1. 需要先有一个publisher实例对象
2. 需要在publisher里面定义好一个生产者的数据生产流：接受数据以后怎么处理这个数据对象
3. 需要有一个subscriber去订阅这个生产者
4. 消费者订阅了这个生产者， 订阅的时候可以根据这个订阅者

二、异步

反应式编程的异步和java的异步编程理念是一致的，都是在接受到数据以后对数据进行处理，不同的是编写这个数据处理流程的方式是不一样的。

- 命令式编程

  命令式编程是通过异步编程的API(nio或者netty)异步接收数据，使用分支控制(if/else), 流程调用(实现类与方法的调用)，循环控制(for/while)等进行构建对这个数据的处理与流转过程。

- 反应式编程

  反应式编程也是需要异步编程的API异步接收数据，不同的是反应式编程以面向对象的方式在一个集中的地方以链式api的写法为这个数据对象定义一个数据处理链，这个数据处理链可以理解为上面说的生产者的流式处理链，而这个生产者接受的元素是由异步的api产生的。



### 3.2 关于反应式编程对比命令式编程说的效率的提升

常说的Reactive steam的异步流式后比传统的写法快了多少的， Reactive steam本质上与与传统的编程方式的核心理念并无二致，都是对数据的处理流程， 两者不同的是对流程处理的编程范式不同。Reactive steam所谓的通过异步能提升的效率使用命令式编程也都是可以实现相等的效果的。

Reactive steam对比命令式编程范式最大的贡献为提供了背压的规范，只要是实现了Reactive Steam规范的库都可以通过背压实现服务端的反馈，传统的命令式编程就是因为没有这一套通用的规范，数据流在进入某个中间件或者某个程序以后，虽然很多内部都声称自己是全异步处理， 实际上进来的网络数据都是同步的，会源源不断的发往你的系统上，最后处理不过来的时候就会堆积在你的内存队列中或者网卡队列中，而如果大家都是实现了Reactive steam规范的包和程序，那就可以通过统一的规范进行背压反馈，服务处理的时间自然就不会因为处理能力不足堆积或者说直接把程序冲垮。 

## 四. 目标

- 管理异步跨边界的数据流交换。即将元素传递到另一个线程或者线程池中。
- 确保接收方不会强制缓冲任意数量的数据， 引入了回压。



## 五. 如何理解

### 5.1 reactor

反应的意思， 就是被动的接受数据，然后对数据做出反应。

### 5.2 Steams

流式处理， 先定义好一条流水线， 然后接收入参，经过流水线加工，到出参。

### 5.3 非阻塞，异步





## 六. 背压

发布者与订阅者之间消息协调，订阅者需要多少数据，发布者产生多少数据，不会造成消息接收者被数据流冲垮。

SpringMVC是同步阻塞模式，数据接收是被动处理，数据会源源不断发送，如果数据处理不了，就会出现消息挤压和堵塞。

![]({{site.url}}/assets/images/back-pressure.png)



## 七. 与jdk1.8 jdk1.9的关系

jdk1.8 支持lamdba表达式和流式处理的api， 但Reactor steam api不是jdk1.8的api的一部分。

*jdk 的steam api是同步的，也支持并行的，但都是阻塞的，而Reactor steam api都是异步非阻塞的*

jdk 1.9原生支持reactor steam api， jdk9 flow包下的内容与reactor steam api一致。



## 八. Reactive steam api具体实现

### 8.1 RxJava

诞生早于 Reactive steam api规范， RxJava 2.0+实现了Reactive steam api规范，但是接口术语略有不同。

### 8.2 Reactor

Pivotal提供的Java实现，它作为Spring Framework 5的重要组成部分，是WebFlux采用的默认反应式框架。

### 8.3 Akka Streams

Akka Streams完全实现了Reactive Streams规范，但Akka Streams API与Reactive Streams API完全分离。

### 8.4 Ratpack

Ratpack是一组用于构建现代高性能HTTP应用程序的Java库。Ratpack使用Java 8，Netty和Reactive原则。可以将RxJava或Reactor与Ratpack一起使用。

### 8.5 Vert.x

Vert.x是一个Eclipse Foundation项目，它是JVM的多语言事件驱动的应用程序框架。Vert.x中的反应支持与Ratpack类似。Vert.x允许我们使用RxJava或其Reactive Streams API的实现。



## 九. 总结

在Reactive Streams api规范出现之前，各种库之间的反应式编程不能互相操作， 导致不兼容。

另外， 反应式编程无法大规模普及的原因是不是所有的库都支持反应式编程， 当反应式编程里面调用了一些同步的api的时，最终的瓶颈也会是在这些同步的api上面。

