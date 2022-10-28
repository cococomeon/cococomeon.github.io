---
layout: post
title: Netty HashWheelTimer
subtitle:
categories: 定时任务
tags: [源码解析]
banner: "/assets/images/banners/home.jpeg"

---

HashWheelTimer功能是添加一个任务，指定在一段时间后执行。只执行一次，不可定时循环执行。

优点：<br>
添加和删除的时间复杂度都是O(1)。添加的时候是直接添加到一个队列中。删除的时候并不是直接移除，而是给这个timeout设置一个cancel的标志位。

缺点：<br>
1. 不能定时重复执行定时任务
2. 单线程执行，如果任务执行时间长会影响后面的任务执行
3. 不适合创建那些时间跨度大的，应为只触发一次，会一直保存在时间轮的bucket中，时间过长会导致链表容易过长，且占用内存。


 ![]({{site.url}}/assets/images/2022-10-27-Netty HashWheelTimer.assets/wheel.jpeg)



## 一. **数据模型**

主要有两个数据结构，一个是构成时间轮的，一个是构成时间轮的每个格子中的

### 1.1 **HashedWheelTimeout**
bucket中的每一个定时任务的封装，形成一个链表。每个定时任务中还有一个round属性
timeout中保存的属性：
1. long类型的deadline，表示的是距离startTime的时间长度
2. remainingRounds，剩余的的转动的轮数，其实上面的deadline一项就足够确定定时任务的执行时间的了。

### 1.2 **HashedWheelBucket**
时间轮中每一个格子的实例，在HashWheelTimer中保存的是HashedWheelBucket数组。数组的下标代表当前格子的跨度id。

### 1.3 **时间轮**
时间轮中保存的状态有：
1. startTime, 时间轮的启动时间，用来计算差值确定具体的执行时间点
2. tick，当前时间轮走过了多少个格子
3. tickDuration，每一个格子的时间跨度是多长




## 二. **线程模型**
内部有一个线程负责像时钟的指针一样，不断地转着。<br>
这个内部线程的执行逻辑：<br>
1. 第三方线程拿到HashWheel实例，然后通过addTimeout(Timetask task)添加定时任务到一个临时queue中
2. HashWheel内部线程worker循环执行，从queue拿出来这个定时任务，并且添加到bucket中
3. 内部线程轮询线程执行


### **2.1 添加与删除定时任务**

```java
import java.util.concurrent.TimeUnit;

public class a {
  public static void main(String[] args) {
    HashWheelTimer timer = new HashWheelTimer();//默认时间轮是100毫秒一格，512格
    TimeOut t = timer.newTimeout(timer, 10, TimeUnit.SECONDS);//添加一个十秒钟后执行的定时任务
    t.cancel();//删除一个定时任务
  }
}

```



### **2.2 时间轮执行主逻辑**
```java
private final class Worker implements Runnable {
    private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

    private long tick;

    @Override
    public void run() {
        // Initialize the startTime.
        startTime = System.nanoTime();
        if (startTime == 0) {
            // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
            startTime = 1;
        }

        // Notify the other threads waiting for the initialization at start().
        startTimeInitialized.countDown();

        do {
            final long deadline = waitForNextTick();//休眠直到到下一个格子，返回醒来后的格子时间与时间轮开始时间的时间差
            if (deadline > 0) {
                int idx = (int) (tick & mask);//求余， 相当于 当前格子数 / 时间轮、定位出当前的格子数处于时间轮的哪个位置
                processCancelledTasks();//处理被取消掉的任务
                HashedWheelBucket bucket = wheel[idx];//取出对应的时间轮上的定时任务联表
                transferTimeoutsToBuckets();//将定时任务队列中的任务添加到时间轮中
                bucket.expireTimeouts(deadline);
                tick++;
            }
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

        // Fill the unprocessedTimeouts so we can return them from stop() method.
        for (HashedWheelBucket bucket : wheel) {
            bucket.clearTimeouts(unprocessedTimeouts);
        }
        for (; ; ) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                break;
            }
            if (!timeout.isCancelled()) {
                unprocessedTimeouts.add(timeout);
            }
        }
        processCancelledTasks();
    }

    private void transferTimeoutsToBuckets() {
    // transfer only max. 100000 timeouts per tick to prevent a thread to stale the workerThread when it just
    // adds new timeouts in a loop.
      for (int i = 0; i < 100000; i++) {
        HashedWheelTimeout timeout = timeouts.poll();
        if (timeout == null) {
          // all processed
          break;
        }
        if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
          // Was cancelled in the meantime.
          continue;
        }

        long calculated = timeout.deadline / tickDuration;//从时间轮开始转动到现在一共要走多少个格子才会触发这个timeout任务
        timeout.remainingRounds = (calculated - tick) / wheel.length;//设置这个定时任务从当前开始还要走多少轮，tick表示时间轮转动到现在已经走了多少个格子了

        //假如我原本是要走187个格子到达执行，但是现在已经到了190个格子了，那就立即把这个定时任务放到当前这个190的格子进行触发
        final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.

        int stopIndex = (int) (ticks & mask);//求余，定位这个定时任务在队列中的位置

        HashedWheelBucket bucket = wheel[stopIndex];
        bucket.addTimeout(timeout);
      }
  }

  public void expireTimeouts(long deadline) {
    HashedWheelTimeout timeout = head;

    // process all timeouts
    while (timeout != null) {//处理这个bucket上的所有定时任务
      HashedWheelTimeout next = timeout.next;
      if (timeout.remainingRounds <= 0) {//轮数已经为0，就是到达自己的这一轮了
        next = remove(timeout);
        if (timeout.deadline <= deadline) {//这里为什么不判断 deadline-timeout.deadline<tickDurtion呢？这样能严格判断这个定时任务是出于这个格子中的。这里实际是考虑到前面的，过了自己执行格子的任务，能够马上执行
          timeout.expire();//执行这个定时任务
        } else {
          // The timeout was placed into a wrong slot. This should never happen.
          throw new IllegalStateException(String.format(
            "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
        }
      } else if (timeout.isCancelled()) {
        next = remove(timeout);//任务取消了移除
      } else {
        timeout.remainingRounds --;//还没有到任务的轮数
      }
      timeout = next;
    }
  }
}
```

## **三. ScheduledExecutorService与HashedWheelTimer的对比？**
ScheduledExecutorService保存任务的时候维护的是一个堆，需要频繁对任务进行添加或者修改的时候，需要频繁修改堆结构导致性能下降。
HashedWheelTimer则不受任务量的限制，也就是定时任务的增删改是很快的，不会额外占用投递线程的时间。它唯一的缺点是，有一个内部线程，在没有定时任务
的时候也会空转占用一定的CPU资源。
