---
layout: post
title: 并发工具 - Reference
subtitle:
categories: Java
tags: [java, juc]
banner: "/assets/images/banners/home.jpeg"

---

一.Reference机制

## **一. Reference类图**

![]({{site.url}}/assets/images/2020-03-04-Reference详细解析.assets/image-20220322180046477.png)

| 类型   | 实现             | 说明                             |
| ------ | ---------------- | -------------------------------- |
| 强引用 |                  | 虚拟机内部实现，有引用就不会被gc |
| 软引用 | softreference    | 内存不足时gc                     |
| 弱引用 | weakreference    | 下一次gc的时候回收               |
| 虚引用 | phantomreference | 不影响引用的对象的生命周期。     |
|        | finalreference   | Object 的finalize()的实现机制    |

 

## **1.1 FinalReference的Finalizer机制**

Finalizer只关联Object的finalize()方法。Finalizer实际就是一个reference， 每一类reference都有一个queue， 当reference被引用的对象被回收的时候，就去处理这个queue中的对象。Finalizer中有一个f-queue，当一个对象没有引用了，会放进这个fqueue里面，然后会执行里面的finilizer方法。这个方法在对象被gc回收前可以进行释放资源， 或者让自己免于被gc的命运。



## **1.2 PhantomReference的Cleaner机制**

PhantomReference不影响对象的gc过程。当reference引用的对象被gc回收的时候， 会将这个**对象的引用(Cleaner)**放进pending链表里面，然后Reference-Handler会处理这个链表里面的cleaner。执行Cleaner中的clean方法进行处理。

## **Cleaner机制和Finializare机制比较**

finializare机制在对象中实现耦合，在对象回收前，给对象一次机会逃逸gc，但不保证执行。

cleaner机制实现与对象解耦，在对象回收后，执行与对象关联的一个runnable。
