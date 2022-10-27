

1. CAS Lock Free
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
