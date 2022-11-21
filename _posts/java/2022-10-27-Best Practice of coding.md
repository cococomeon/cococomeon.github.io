---
layout: post
title: Best Practice of coding
subtitle:
categories: Coding
tags: []
banner: "/assets/images/banners/home.jpeg"
---

## CAS Lock Free
当某一个属性可能会被并发更新的时候，我们可以通过锁的方式进行同步，避免出现多线程问题。但是这个锁的粒度是在应用层的，粒度大。
因此我们可以借助java的cas特性，在**应用层无需加锁**也能保证属性的多线程并发的安全性。

我们可以使用JUC中的中的Updater类，将某一个类中的某个属性封装成一个Updater对象，多线程更新的这个属性的时候，可以
直接使用这个Updater进行更新就行。下面的实例代码就是使用java中的cas机制，全程没有应用层的lock操作。
```java
public class demo {
  private volatile int state; //
  public static final Integer INIT = 0;
  public static final Integer START = 1;
  public static final Integer STOP = 2;

  AtomicIntegerFieldUpdater STATE_FIELD_UPDATER = AtomicIntegerFieldUpdater.newUpdater(this, "state");

  public void start(){
    switch(STATE_FIELD_UPDATER.get(this)) {
    case INIT:
        if(STATE_FIELD_UPDATER.compareAndSet(this, INIT, START)){ //这里其实是在CPU核心级别加了锁，并不是完全的无锁，而是将锁的粒度降到了最小，小到在应用层面来说就是没有锁
            //执行启动,上面通过多核之间的并发锁，保证多线程的有序性，一个核心操作这个属性的时候吗，肯定是在上一个线程操作之后。
        }
    case START:
        //
    case STOP:
        break;
    }

  }
}
```


适用场景：某个属性需要被并发更新，并且更新动作不影响后续线程的操作。就是这个动作哪个线程发起都可以，并且别的线程不会收到你这个动作的影响。<br>
不适用场景：线程之间存在时序要求，比如"懒初始化"策略，多个线程去获取一个数据，只需要一个线程去初始化，其他线程等待初始化完成就行的场景。

CAS具体的典型场景：
1. 根据某个状态做一些后台动作，例如起一个后台线程做一些事情
2. 后台守护线程，循环退出的状态标志位。


## 位运算求余
先说结论与要求：当长度len等于2的n次幂时，a & (len-1) = a % len。

论证：
len为2的n次幂的时候，<br>
`len`的二进制形如：0000 1000, <br>
`len-1`二进制形如：0000 0111。<br>
由于后面的位都是1，与其做与运算的时候肯定小于len，符合除法规则

```java
//如何获取一个数最近的二次幂
public class demo{
  static final int tableSizeFor(int cap) { //jdk hashmap中的方法
    int n = cap -1;
    n |= n>>>1;   //将最高位的1复制1个，得到2个1
    n |= n>>>2;		//将上面得到的2个1复制，得到4个1
    n |= n>>>4;		//将上面得到的4个1复制，得到8个1
    n |= n>>>8;		//将上面得到的8个1复制，得到16个1
    n |= n>>>16;	//将上面得到的16个1复制，得到32个1
    return cap;
  }

  private static int normalizeTicksPerWheel(int ticksPerWheel) { //xxl-job中的方法
    int normalizedTicksPerWheel = 1;
    while (normalizedTicksPerWheel < ticksPerWheel) {
        normalizedTicksPerWheel <<= 1;
    }
    return normalizedTicksPerWheel;
  }
}
```



