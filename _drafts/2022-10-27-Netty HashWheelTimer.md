
HashWheelTimer功能是添加一个任务，指定在一段时间后执行。只执行一次，不可定时循环执行。

优点：<br>
添加和删除的时间复杂度都是O(1)。添加的时候是直接添加到一个队列中。删除的时候并不是直接移除，而是给这个timeout设置一个cancel的标志位。

缺点：<br>
1. 不能定时重复执行定时任务
2. 单线程执行，如果任务执行时间长会影响后面的任务执行
3. 不适合创建那些时间跨度大的，应为只触发一次，会一直保存在时间轮的bucket中，时间过长会导致链表容易过长，且占用内存。


 ![]({{site.url}}/2022-10-27-Netty HashWheelTimer.assets/wheel.jpeg)



## 一. **数据模型**

主要有两个数据结构，一个是构成时间轮的，一个是构成时间轮的每个格子中的

### 1.1 **HashedWheelTimeout**
bucket中的每一个定时任务的封装，形成一个链表。每个定时任务中还有一个round属性

### 1.2 **HashedWheelBucket**
时间轮中每一个格子的实例，在HashWheelTimer中保存的是HashedWheelBucket数组。数组的下标代表当前格子的跨度id。



## 二. **线程模型**
1. 第三方线程拿到HashWheel实例，然后通过addTimeout(Timetask task)添加定时任务到一个临时queue中
2. HashWheel内部线程worker循环执行，从queue拿出来这个定时任务，并且添加到bucket中
3. 内部线程轮询线程执行


添加与删除定时任务

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



将定时任务添加进时间轮


轮询时间轮执行定时任务





## ScheduledExecutorService与HashedWheelTimer的对比？
ScheduledExecutorService保存任务的时候维护的是一个堆，需要频繁对任务进行添加或者修改的时候，需要频繁修改堆结构导致性能下降。
HashedWheelTimer则不受任务量的限制，也就是定时任务的增删改是很快的，不会额外占用投递线程的时间。它唯一的缺点是，有一个内部线程，在没有定时任务
的时候也会空转占用一定的CPU资源。
