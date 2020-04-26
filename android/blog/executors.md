## ThreadPoolExecutor

可以使用ThreadPoolExecutor创建一个线程池，其作用如下：

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

参数作用如下：

1. corePoolSize：核心线程数，默认情况下线程池是空的，只有任务提交时才会创建线程，如果当前运行的线程数少于corePoolSize，则会创建新线程来处理任务，如果等于或多于corePoolSize，则不再创建。可以调用prestartAllCoreThreads方法来提前创建并启动所有核心线程来等待任务。
2. maximumPoolSize：想城池允许创建的最大线程数，如果任务队列满了并且线程数小于maximumPoolSize时，则线程池仍旧会创建新的线程来处理任务。
3. keepAliveTime：非核心线程闲置的超时时间。
4. unit：参数的时间单位。
5. workQueue：如果当前线程数大于corePoolSize，则将此任务添加到任务队列中。
6. threadFactory：
7. handler：饱和策略，当任务队列和线程池都满了时所采取的应对策略。

## 线程池处理流程和原理

1. 提交任务后，线程池先判断线程数是否达到了核心线程数，如果未达到核心线程数，则创建核心线程处理任务，否则，执行下一步。
2. 判断任务队列是否满了，如果没满，则将任务添加到任务队列中，否则，执行下一步。
3. 因为任务队列满了，线程池就判断线程数是否达到了最大线程数，如果未达到，则创建非核心线程处理任务，否则，执行饱和策略，默认会抛出RejectedExecutionException异常。

## 线程池种类

Executors类提供了四个方法创建四种线程池，分别是：FixedThreadPool、CachedThreadPool、SingleThreadExecutor和ScheduledThreadPool。

### FixedThreadPool

其创建方法如下：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

corePoolSize和maximumPoolSize都设置为制定的参数nThreads，，表明FixedThreadPool只有核心线程，keepAliveTime为0表明多于的线程会被立即终止，同时任务队列采用了无界阻塞队列。

### CachedThreadPool

其创建方法如下：

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

其corePoolSize为0，maximumPoolSize为Integer.MAX_VALUE，表明，它没有核心线程，且非核心线程无界，keepAliveTime为60秒，即空闲线程等待新任务的最长时间为60s。同时，它使用了不存储元素的阻塞队列SynchronousQueue。

当执行execute方法时，首先会执行SynchronousQueue的offer方法提交任务，如果有空闲的线程在等待任务，则将任务交由次线程处理，否则，将创建新线程处理此任务。当线程处理完成时，会调用SynchronousQueue的poll方法来等待新任务，若超过了60秒还没有等到新任务，则会将此线程终止。因为maximumPoolSize无界，所以如果提交任务的速度大于线程池中线程处理任务的速度，就会不断创建新线程。

CachedThreadPool适合大量需要立即处理并且耗时较少的任务。

### SingleThreadExecutor

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

其中除了corePoolSize和maximumPoolSize参数都为1外，其他参数都与FixedThreadPool相同。

### ScheduledThreadPool

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }

public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```

