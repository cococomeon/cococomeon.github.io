---
layout: post
title: select, poll与epoll
subtitle:
categories: Netty
tags: [epoll]
banner: "/assets/images/banners/home.jpeg"

---

netty 在构建服务器线程池的时候需要指定parent线程池和child线程池，parent负责端口监听，一旦有新链接就注册到child连接池上，该连接的IO操作都由这个线程完成。

也就是这个IO线程会被多个连接注册，一个IO线程负责多个连接的IO时间，也就是多路复用。Netty底层提供select或者epoll来实现多路复用。



## 一. select 

服务端每建立一个连接都相当于打开一个一个文件，获得这个文件的描述符(fd)，相同的四元组（源ip/源port/目标ip/目标port）对应同一个fd。select和poll有相似的地方，不一样的是select使用的是数组，有连接数限制，poll使用的是链表没有连接数限制。

从C源码调用角度看select：

1. 构建三个fd数组，分别关注读/写/异常事件。
2. 传入这三个数组，调用系统的select方法，将数组传到内核态，等待部分fd就绪后，把fd数组传回给用户态
3. 用户程序调用fd数组进行遍历，处理就绪的fd

缺点：

1. 每次都要传入大量的fd，导致大量的用户态和内核态的拷贝。
2. 线性扫描的轮询，效率低下

poll与select类似，差别在于poll是链表，没有最大文件数限制，还有就是poll是水平触发的。

select 文件描述符的的大小限制为1024？这在内核中的源码定义了

```c
/linux/posix_types.h:
#define __FD_SETSIZE         1024
```

select 和 poll都是需要轮询的查询整个fd_set, 当连接数上来以后， 例如十万连接，活跃的只有100， 这个时候select就会显得力不从心了。





## 二. epoll

epoll主要有三个函数构成

1. epoll_create()创建一个epoll实例
2. epoll_ctl()，增加或者删除一个新链接至epoll红黑树中, 并且设置回调函数
3. epoll_wait()，一旦fd某个事件准备就绪，就会把这个fd与就绪事件放到就绪链表中，并唤醒等待队列中的线程。



### 2.1 调用过程

![在这里插入图片描述]({{site.url}}/assets/images/select,poll,epoll.png)

1. 通过epoll_create()创建eventpoll对象实例，并且返回eventpoll对象的文件句柄
2. 通过epoll_ctl()把服务端监听的句柄封装成epitem，添加至eventpoll对象的红黑树中进行管理
3. 调用epoll_wait()函数等待被监听的文件句柄发生变化
4. 当监听的文件句柄发生变化的时候，内核会通过调用回调函数将epitem添加到eventpoll对象的就绪列表rdlist中，并且把eventpoll对象中的就绪队列的文件列表复制到epoll_wait()函数中的events函数中
5. 唤醒epoll_wait()函数，让应用层进行处理对应的文件句柄。



### 2.2. 调用细节

#### 2.2.1 epoll_create

函数定义

```C
#include <sys/epoll.h>
// 创建一棵红黑树
int epoll_create(int size);
  	参数: 
  		size: 没意义（参数 size 是由于历史原因遗留下来的，现在不起作用）
  	返回值;
  		>0: epoll句柄
  		<=0: 失败
```

主要工作内容：创建一个eventpoll对象。

```C
/*
 * eventpoll对象数据结构
 */
struct eventpoll {
	/*...省略...*/

	/* 准备就绪的文件句柄集合 */
	struct list_head rdllist;

	/* 红黑树根节点 */
	struct rb_root_cached rbr;

	/*...省略...*/
};

/*
 * eventpoll中的红黑树节点包装
 */
struct epitem {
	union {
		/* RB tree node links this structure to the eventpoll RB tree */
		struct rb_node rbn;
		/* Used to free the struct epitem */
		struct rcu_head rcu;
	};

	/* 关联eventpoll中的就绪列表 */
	struct list_head rdllink;

	/*
	 * Works together "struct eventpoll"->ovflist in keeping the
	 * single linked chain of items.
	 */
	struct epitem *next;

	/* 发生的事件信息 */
	struct epoll_filefd ffd;

	/* List containing poll wait queues */
	struct eppoll_entry *pwqlist;

	/* 红黑树容器 */
	struct eventpoll *ep;

	/* wakeup_source used when EPOLLWAKEUP is set */
	struct wakeup_source __rcu *ws;

	/* 描述感兴趣的事件与原文件句柄描述符 */
	struct epoll_event event;
};
```





#### 2.2.2 epoll_ctl

函数定义

```C
// 对epoll树进行管理: 添加节点, 删除节点, 修改已有的节点属性
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
  	参数:
  		- epfd: epoll_create的返回的epoll句柄
  		- op: 要进行什么样的操作
  			EPOLL_CTL_ADD: 注册新节点, 添加到红黑树上
  			EPOLL_CTL_MOD: 修改检测的文件描述符的属性
  			EPOLL_CTL_DEL: 从红黑树上删除节点
  		- fd: 要检测的文件描述符的值
  		- event: 检测文件描述符的什么事件
```

主要工作内容：对eventpoll对象里面的红黑树节点进行增删改查

工作逻辑：

1. 创建socket事件对应的epitem对象
2. 将epitem添加至socket等待队列中，并设置唤醒函数ep_poll_callback()函数，当socket状态发生变更的时候，调用这个回调函数。回到函数如何和这个epitem对应起来的？
3. 回调函数主要是把这个epitem添加至eventpoll对象的就绪队列rdlist中，然后唤醒epoll_wait中等待的进程或线程。



#### 2.2.3 epoll_wait



### 2.3 epoll的两种触发

#### 2.3.1 水平触发 LT(level triggered)
缺省的工作方式，并且同时支持block（阻塞）和no-block （非阻塞）socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后可以对这个就绪的fd进行IO操作。只要有数据，内核会一直通知。

#### 2.3.2 边缘触发 ET(edge-triggered)
高速工作方式，只支持no-block（非阻塞） socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll通知。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了。但是请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)。
ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高`。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死











java的select是对epoll， poll， select的封装，屏蔽了底层细节。







[Netty关于select/epoll](https://blog.csdn.net/lblblblblzdx/article/details/88795242)

[彻底搞懂文件描述符/文件句柄/文件指针的区别与联系](https://www.jianshu.com/p/ad879061edb2)

[select、poll、epoll之间的区别(搜狗面试)](https://www.cnblogs.com/aspirant/p/9166944.html)

[NIO的epoll空轮询bug](https://blog.51cto.com/u_9058648/3563866)

[Linux下的I/O复用与epoll详解](https://www.cnblogs.com/lojunren/p/3856290.html)

[select，poll和epoll详解](https://blog.csdn.net/u010306832/article/details/119942290)
