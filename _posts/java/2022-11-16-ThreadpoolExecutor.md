

![img.png]({{site.url}}/assets/images/2022-11-16-ThreadpoolExecutor.assets/source.png)


![img.png]({{site.url}}/assets/images/2022-11-16-ThreadpoolExecutor.assets/img_1.png)


|接口/类|说明|
|---|---|
|Executor|顶级执行器接口|
|ExecutorService|线程池封装接口，封装了线程池的关闭，提交任务，批量执行的方法|
|AbstractExecutorService|线程池的骨架实现，主要负责将任务包装成future，然后执行执行器方法|
|ThreadPoolExecutor|线程池默认实现，可在此基础上进行封装，并留下了|


**【ExecutorService】**
主要定义一个执行器服务应该包含销毁方法，提交任务方式，批量执行机制。

|方法|说明|
|---|---|
|shutdown()/shutdownNow()/isShutdown()/isTerminated()/awaitTermination(timeout, unit)|关闭线程池|
|submit(Callable)/submit(callable, result)/submit(runnable)|提交任务的方法|
|invokeAll(tasks)/invokeAll(tasks, timeout, unit)/invokeAny(tasks)/invokeAny(tasks, timeout, unit)|多线程批量任务处理|


**【AbstractExecutorService】**
实现了一个执行器需要包含的抽象骨架，主要负责将任务封装成异步的future，再提交给具体的线程池执行。

|方法|说明|
|---|---|
|newTaskfor(runnable)/newTaskfor(callable)|将任务封装成future返回|
|submit(runnable)/submit(runnable, result)/submit(callable)|将任务task封装成future，然后执行execute()方法执行这个task。|
|doInvokeAny(list<Runnable>)/invokeAll/invokeAny|将任务task封装成future，然后执行|

```java
public abstract class AbstractExecutorService{
  /**
   * 启动多个线程池的所有线程执行给定的任务，只要其中一个线程任务完成，其他线程的任务便取消，同时返回结果。
   *
   * ExecutorCompletionService功能：
   *   主要是将当前的线程池进行了一层封装，内部使用一个阻塞队列，只要某一个线程执行完毕就往这个队列里面塞结果，
   *   只要这个队列里面拿到一个结果就算完成了这一次的invoke动作。
   */
  private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks, boolean timed, long nanos)
    throws InterruptedException, ExecutionException, TimeoutException {
   //略
    ArrayList<Future<T>> futures = new ArrayList<>(ntasks);
    //封装一下当前骨架对象， ExecutorCompletionService：内部包了一个阻塞队列，完成的任务就往这个队列上面塞
    ExecutorCompletionService<T> ecs =
      new ExecutorCompletionService<T>(this);


    try {

      ExecutionException ee = null;
      final long deadline = timed ? System.nanoTime() + nanos : 0L;
      Iterator<? extends Callable<T>> it = tasks.iterator();

      // Start one task for sure; the rest incrementally
      futures.add(ecs.submit(it.next()));
      --ntasks;
      int active = 1;

      for (;;) {
        Future<T> f = ecs.poll();
        if (f == null) {
          if (ntasks > 0) {
            --ntasks;
            futures.add(ecs.submit(it.next()));
            ++active;
          }
          else if (active == 0)
            break;
          else if (timed) {
            f = ecs.poll(nanos, NANOSECONDS);
            if (f == null)
              throw new TimeoutException();
            nanos = deadline - System.nanoTime();
          }
          else
            f = ecs.take();
        }
        if (f != null) {
          --active;
          try {
            return f.get();
          } catch (ExecutionException eex) {
            ee = eex;
          } catch (RuntimeException rex) {
            ee = new ExecutionException(rex);
          }
        }
      }

      if (ee == null)
        ee = new ExecutionException();
      throw ee;

    } finally {
      cancelAll(futures);
    }
  }
}
```

## **ThreadpoolExecutor**
具体的线程池实现，丰富了线程池的具体的特性。





## 线程池的线程模型
1. 内部通过一个BlockingQueue，阻塞队列的方式存放任务。
2. 所有的线程都有一个while循环，在这个循环不断地去取这个队列中的任务出来执行。
3. 提交任务的线程会参与创建工作线程，投递任务，修改线程池状态等工作。

**【线程工作的循环】**



**【任务提交与执行】**






线程池的最佳实践
1. 后台任务的线程池需要保证线程执行有兜底，不能让线程因为任务的异常执行结果而退出
2. 手动创建线程池
