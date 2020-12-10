# Rxjava解析

rxjava是基于事件流的异步编程框架，在Android开发中受到广泛使用。

## 基本用法

首先，介绍一下它的基本使用方法：

先在build.gradle中添加引用：

```
implementation 'io.reactivex.rxjava2:rxjava:2.2.9'
implementation 'io.reactivex.rxjava2:rxandroid:2.1.1'
```



```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        Log.d(TAG, "subscribe: current thread " + Thread.currentThread().getName());
        for (int i = 0; i < 5; i++) {
            emitter.onNext(i);
        }
        emitter.onComplete();
    }
})
    .subscribeOn(Schedulers.computation())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Observer<Integer>() {
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe: thread " + Thread.currentThread().getName());
        }

        @Override
        public void onNext(Integer integer) {
            Log.d(TAG, "onNext: thread " + Thread.currentThread().getName());
            Log.d(TAG, "onNext: " + integer);
        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onComplete() {
            Log.d(TAG, "onComplete: thread " + Thread.currentThread().getName());
        }
    });
```

首先，实现ObservableOnSubscribe接口，创建一个被观察的对象，将其传给Observable.create()方法，在这个被观察的对象中可以执行一些耗时任务，比如计算任务或者io任务。

然后通过subscribeOn()方法来指定被观察执行的线程，rxjava提供了五种线程，分别是`Schedulers.computation()`、`Schedulers.io()`、Schedulers.newThread()`、Schedulers.single()`、`Schedulers.trampoline()`。基于Android平台，可以引入rxandroid，使用`AndroidSchedulers.mainThread()`即可指定Android主线程运行了。

在通过observeOn()方法指定结果回调运行的线程。

最后通过subscribe()指定观察者。

## 订阅实现原理

前面的demo是rxjava的基础用法，但是我们可以再简化一点，先把线程切换的逻辑去掉，这样就是一个最小的示例了，如下：

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        Log.d(TAG, "subscribe: current thread " + Thread.currentThread().getName());
        for (int i = 0; i < 5; i++) {
            emitter.onNext(i);
        }
        emitter.onComplete();
    }
})
    .subscribe(new Observer<Integer>() {
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe: thread " + Thread.currentThread().getName());
        }

        @Override
        public void onNext(Integer integer) {
            Log.d(TAG, "onNext: thread " + Thread.currentThread().getName());
            Log.d(TAG, "onNext: " + integer);
        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onComplete() {
            Log.d(TAG, "onComplete: thread " + Thread.currentThread().getName());
        }
    });
```

现在根据这个demo来分析一下rxjava的实现原理。

先看一下Observable.create()的实现：

```java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

其中`RxJavaPlugins.onAssembly()`是用来调用hook函数的，相当于这里return的就是`new ObservableCreate<T>(source)`，也就是说，在这里用`ObservableCreate`自定义的`ObservableOnSubscribe`对象封装起来了。

接下去再看一下`subscribe()`实现:

```java
@SchedulerSupport(SchedulerSupport.NONE)
@Override
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");

        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        // can't call onSubscribe because the call might have set a Subscription already
        RxJavaPlugins.onError(e);

        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
```

`subscribe`是`Observable`中的final方法，其中调用了`subscribeActual(observer)`实现真正的订阅逻辑，`subscribeActual`是一个抽象方法，它在子类中实现：

```java
protected abstract void subscribeActual(Observer<? super T> observer);
```

由于`create`中返回的是`ObservableCreate`的示例，因此我们看一下此类的`subscribeActual`方法：

```java
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);

    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```

先把用户传进来的observer封装成一个CreateEmitter对象，然后调用observer.onSubscribe()方法通知用户，接着调用`source.subscribe(parent);`这里的source就是之前实现的`ObservableOnSubscribe`匿名对象:

```java
public ObservableCreate(ObservableOnSubscribe<T> source) {
    this.source = source;
}

public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

到这里，就回到了用户实现的匿名内部类中，用户可以再匿名内部类中调用onNext()、onCompete()等传递时间处理结果。这里贴一下订阅的序列图：

![image-20200624143730199](.\imgs\image-20200624143730199.png)

## 操作符使用及实现原理

rxjava的强大之处在于它支持各种各样的操作符，现在我们来看看如何使用它的操作符吧：

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        Log.d(TAG, "subscribe: current thread " + Thread.currentThread().getName());
        for (int i = 0; i < 5; i++) {
            emitter.onNext(i);
        }
        emitter.onComplete();
    }
})
    .map(new Function<Integer, String>() {
        @Override
        public String apply(Integer integer) throws Exception {
            return "emit" + integer;
        }
    })
    .subscribe(new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe: thread " + Thread.currentThread().getName());
        }

        @Override
        public void onNext(String integer) {
            Log.d(TAG, "onNext: thread " + Thread.currentThread().getName());
            Log.d(TAG, "onNext: " + integer);
        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onComplete() {
            Log.d(TAG, "onComplete: thread " + Thread.currentThread().getName());
        }
    });
```

demo中使用map操作符将一个整数转换成字符串，可以看出，其使用方法就是在create和subscribe之前使用map()函数对数据进行转换，其中map()接受一个Function接口的对象，用户在实现类中根据需要实现自己的转换逻辑，那么接下来就看看map函数是如何实现的：

map()函数是`Observable`中的final方法：

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```

这里使用用户传入的mapper转换函数和当前的`Observable`对象（就是`ObservableCreate`的实例）构造一个`ObservableMap`对象，因此需要分析`ObservableMap`的实现逻辑：

上一节分析了订阅的实现原理，知道最后会调用Observable子类的subscribeActual()方法来实现具体的订阅逻辑，因此我们直接跳到`ObservableMap`的subscribeActual()方法：

```java
@Override
public void subscribeActual(Observer<? super U> t) {
    source.subscribe(new MapObserver<T, U>(t, function));
}
```

这里新建了一个MapObserver对象，并将其传给上一级的`Observable`，即`ObservableCreate`，然后交由`ObservableCreate`进行处理，其逻辑已经在之前讲过了，这里看一下`MapObserver`实现：

```java
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }

    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != NONE) {
                downstream.onNext(null);
                return;
            }

            U v;

            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            downstream.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Nullable
        @Override
        public U poll() throws Exception {
            T t = qd.poll();
            return t != null ? ObjectHelper.<U>requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
    }
}
```

MapObserver是ObservableMap的静态内部类，在其构造器中，MapObserver保存了用户定义的Function对象。MapObserver重写了Observer的onNext对象，在其中调用了mapper转换函数进行数据转换。

到这里，map操作符的实现原理也就清晰了，就是在subscribe之前添加map()函数，将`ObservableCreate`转换成`ObservableMap`对象，然后在`ObservableMap`中创建新的MapObserver实现映射逻辑。

## 链式调用实现原理

rxjava的一大特性就是链式调用，这其实是每个不同的接口都返回相应的Observable对象实现的，看一下下面的例子：

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        Log.d(TAG, "subscribe: current thread " + Thread.currentThread().getName());
        for (int i = 0; i < 5; i++) {
            emitter.onNext(i);
        }
        emitter.onComplete();
    }
})
    .map(new Function<Integer, String>() {
        @Override
        public String apply(Integer integer) throws Exception {
            return "emit" + integer;
        }
    })
    .subscribeOn(Schedulers.computation())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe: thread " + Thread.currentThread().getName());
        }

        @Override
        public void onNext(String integer) {
            Log.d(TAG, "onNext: thread " + Thread.currentThread().getName());
            Log.d(TAG, "onNext: " + integer);
        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onComplete() {
            Log.d(TAG, "onComplete: thread " + Thread.currentThread().getName());
        }
    });
```

其中create、map实现前面都分析过，他们分别返回的是`ObservableCreate`和`ObservableMap`的实例，再看看subscribeOn和observeOn:

```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}

public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```

可以看到分别返回的是ObservableSubscribeOn和ObservableObserveOn对象，那么我们分别来看看他们的类结构：

```java
public final class ObservableCreate<T> extends Observable<T> {}

public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {}

public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {}

public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {}

abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {

    /** The source consumable Observable. */
    protected final ObservableSource<T> source;

    /**
     * Constructs the ObservableSource with the given consumable.
     * @param source the consumable Observable
     */
    AbstractObservableWithUpstream(ObservableSource<T> source) {
        this.source = source;
    }

    @Override
    public final ObservableSource<T> source() {
        return source;
    }

}
```

可以看出，除了ObservableCreate直接继承Observable外，其他类都是通过AbstractObservableWithUpstream间接Observable的。由此可以看出，rxjava的链式调用是通过返回一系列继承了AbstractObservableWithUpstream的对象来实现的，大体类结构如下所示：

![image-20200624155618624](.\imgs\image-20200624155618624.png)

## 线程切换原理

回到开头的demo中：

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        Log.d(TAG, "subscribe: current thread " + Thread.currentThread().getName());
        for (int i = 0; i < 5; i++) {
            emitter.onNext(i);
        }
        emitter.onComplete();
    }
})
    .subscribeOn(Schedulers.computation())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Observer<Integer>() {
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe: thread " + Thread.currentThread().getName());
        }

        @Override
        public void onNext(Integer integer) {
            Log.d(TAG, "onNext: thread " + Thread.currentThread().getName());
            Log.d(TAG, "onNext: " + integer);
        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onComplete() {
            Log.d(TAG, "onComplete: thread " + Thread.currentThread().getName());
        }
    });
```

这里调用了subscribeOn和observeOn来切换被观察者和观察者的运行线程，先看一下subscribeOn的实现：

```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```

这里创建了ObservableSubscribeOn对象来处理相应的线程切换请求，同时把处理线程scheduler传给它，我们看一下它的subscribeActual方法：

```java
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

    observer.onSubscribe(parent);

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```

这里使用scheduler.scheduleDirect()将当前任务放到scheduler指定的线程中执行：

这一部分的时序如下：

![image-20200625145906029](.\imgs\image-20200625145906029.png)

接下来再看看observeOn实现：

```java
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```

可以看到，实现都交给了ObservableObserveOn，因此直接查看ObservableObserveOn

的subscribeActual()：

```java
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker();

        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}
```

可以看出，大部分情况下，都是走else分支的，也就是基本都是通过scheduler.createWorker();先创建一个Worker，然后将其传递到ObserveOnObserver中进行处理：

```java
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
	...

    static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {
	...

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            }
            schedule();
        }

        @Override
        public void onError(Throwable t) {
            if (done) {
                RxJavaPlugins.onError(t);
                return;
            }
            error = t;
            done = true;
            schedule();
        }

        @Override
        public void onComplete() {
            if (done) {
                return;
            }
            done = true;
            schedule();
        }

        void schedule() {
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }

        void drainNormal() {
            int missed = 1;

            final SimpleQueue<T> q = queue;
            final Observer<? super T> a = downstream;

            for (;;) {
                if (checkTerminated(done, q.isEmpty(), a)) {
                    return;
                }

                for (;;) {
                    boolean d = done;
                    T v;

                    try {
                        v = q.poll();
                    } catch (Throwable ex) {
                        Exceptions.throwIfFatal(ex);
                        disposed = true;
                        upstream.dispose();
                        q.clear();
                        a.onError(ex);
                        worker.dispose();
                        return;
                    }
                    boolean empty = v == null;

                    if (checkTerminated(d, empty, a)) {
                        return;
                    }

                    if (empty) {
                        break;
                    }

                    a.onNext(v);
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

        void drainFused() {
            int missed = 1;

            for (;;) {
                if (disposed) {
                    return;
                }

                boolean d = done;
                Throwable ex = error;

                if (!delayError && d && ex != null) {
                    disposed = true;
                    downstream.onError(error);
                    worker.dispose();
                    return;
                }

                downstream.onNext(null);

                if (d) {
                    disposed = true;
                    ex = error;
                    if (ex != null) {
                        downstream.onError(ex);
                    } else {
                        downstream.onComplete();
                    }
                    worker.dispose();
                    return;
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

        @Override
        public void run() {
            if (outputFused) {
                drainFused();
            } else {
                drainNormal();
            }
        }
		...
    }
}
```

可以看出，在回调到Observer的onNext、onComplete等回调函数时，都通过schedule()方法将任务通过worker.schedule()调度掉指定的线程中，然后再run方法中调用用户的回调函数通知用户。

其处理逻辑如下：

![image-20200624163207366](.\imgs\image-20200624163207366.png)

### Scheduler

前面分析subscribeOn()和observeOn()时都接收Scheduler对象，最终也都是通过Scheduler进行线程间的切换，所以接下来要再来详细分析一下Scheduler实现了。

首先，看一下Scheduler的类结构：

![image-20200625155907485](.\imgs\image-20200625155907485.png)

先来介绍一下Scheduler和Worker之间的关系，一般来说，Worker提供具体的线程池操作，也就是说通过Scheduler提交的task最终基本上都是交由Worker运行，而Scheduler子类则根据各自的特性来创建合适的线程池以供调度，比如ComputationScheduler就会根据cpu数目来创建适合的线程池数量，以尽量提高处理器的效率。下面分别介绍一下这些调度器各自的特性。

| Scheduler            | 说明                               |
| -------------------- | ---------------------------------- |
| ComputationScheduler | 计算密集型任务调度器               |
| IOScheduler          | IO 密集型任务调度器                |
| TrampolineScheduler  | 在某个调用 schedule 的线程执行     |
| NewThreadScheduler   | 每个 Worker 对应一个新线程         |
| SingleScheduler      | 所有 Worker 使用同一个线程执行任务 |
| ExecutorScheduler    | 使用 Executor 作为任务执行的线程   |
| HandlerScheduler     | 用于切换到android主线程中执行      |

从Scheduler的类结构中可以看出其中包含scheduleDirect()方法和createWorker()方法，createWorker()用于创建Worker对象，用户在之后手动调用Worker对象的schedule()方法执行任务，shceduleDirect(）方法如何实现的呢？在基类Scheduler中，其默认实现如下：

```java
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```

首先新建一个Worker对象，然后把任务封装成一个DisposeTask对象，调用worker.schedule()执行。

接下去就需要分析一下Scheduler子类中是如何实现createWorker()方法的了，这里就挑ComputationScheduler来分析吧。

#### ComputationScheduler

先来看一下ComputationScheduler的createScheduler()：

```java
public Worker createWorker() {
    return new EventLoopWorker(pool.get().getEventLoop());
}
```

首先，这里用到了pool，我们需要看一下pool是个什么类型的变量，再看一下pool.get().getEventLoop()返回的是怎样的值，这就需要从ComputationScheduler的创建说起了。

先看看pool的定义和初始化：

```java
final AtomicReference<FixedSchedulerPool> pool;

public ComputationScheduler(ThreadFactory threadFactory) {
    this.threadFactory = threadFactory;
    this.pool = new AtomicReference<FixedSchedulerPool>(NONE);
    start();
}
```

从这里可以看出pool是FixedSchedulerPool的原子引用，那么FixedSchedulerPool是做什么用的呢？其实这是一个Worker对象的缓存池，接下去看start()方法的实现：

```java
public void start() {
    FixedSchedulerPool update = new FixedSchedulerPool(MAX_THREADS, threadFactory);
    if (!pool.compareAndSet(NONE, update)) {
        update.shutdown();
    }
}
```

这里使用MAX_THREADS变量新建了一个FixedSchedulerPool对象，将其保存为原子引用，也就是创建了一个FixedSchedulerPool大小的Worker缓存池，而MAX_THREADS的值究竟是什么，我们来看看：

```java
/** The maximum number of computation scheduler threads. */
static final int MAX_THREADS;

static {
    MAX_THREADS = cap(Runtime.getRuntime().availableProcessors(), 			Integer.getInteger(KEY_MAX_THREADS, 0));
	......
}

static int cap(int cpuCount, int paramThreads) {
    return paramThreads <= 0 || paramThreads > cpuCount ? cpuCount : paramThreads;
}
```

从这里不难看出，默认情况下MAX_THREADS就是CPU的核心数，接下去看一看FixedSchedulerPool的构造器：

```java
FixedSchedulerPool(int maxThreads, ThreadFactory threadFactory) {
    // initialize event loops
    this.cores = maxThreads;
    this.eventLoops = new PoolWorker[maxThreads];
    for (int i = 0; i < maxThreads; i++) {
        this.eventLoops[i] = new PoolWorker(threadFactory);
    }
}
```

这里根据传进来的maxThreads来创建PoolWorker数组，这就是Worker的缓存池。我们看一下它的构造函数：

```java
static final class PoolWorker extends NewThreadWorker {
    PoolWorker(ThreadFactory threadFactory) {
        super(threadFactory);
    }
}
```

PoolWorker继承了NewThreadWorker，看看它的实现：

```java
public class NewThreadWorker extends Scheduler.Worker implements Disposable {
    private final ScheduledExecutorService executor;

    volatile boolean disposed;

    public NewThreadWorker(ThreadFactory threadFactory) {
        executor = SchedulerPoolFactory.create(threadFactory);
    }

    @NonNull
    @Override
    public Disposable schedule(@NonNull final Runnable run) {
        return schedule(run, 0, null);
    }

    @NonNull
    @Override
    public Disposable schedule(@NonNull final Runnable action, long delayTime, @NonNull TimeUnit unit) {
        if (disposed) {
            return EmptyDisposable.INSTANCE;
        }
        return scheduleActual(action, delayTime, unit, null);
    }

    /**
     * Schedules the given runnable on the underlying executor directly and
     * returns its future wrapped into a Disposable.
     * @param run the Runnable to execute in a delayed fashion
     * @param delayTime the delay amount
     * @param unit the delay time unit
     * @return the ScheduledRunnable instance
     */
    public Disposable scheduleDirect(final Runnable run, long delayTime, TimeUnit unit) {
        ScheduledDirectTask task = new ScheduledDirectTask(RxJavaPlugins.onSchedule(run));
        try {
            Future<?> f;
            if (delayTime <= 0L) {
                f = executor.submit(task);
            } else {
                f = executor.schedule(task, delayTime, unit);
            }
            task.setFuture(f);
            return task;
        } catch (RejectedExecutionException ex) {
            RxJavaPlugins.onError(ex);
            return EmptyDisposable.INSTANCE;
        }
    }

    /**
     * Schedules the given runnable periodically on the underlying executor directly
     * and returns its future wrapped into a Disposable.
     * @param run the Runnable to execute in a periodic fashion
     * @param initialDelay the initial delay amount
     * @param period the repeat period amount
     * @param unit the time unit for both the initialDelay and period
     * @return the ScheduledRunnable instance
     */
    public Disposable schedulePeriodicallyDirect(Runnable run, long initialDelay, long period, TimeUnit unit) {
        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        if (period <= 0L) {

            InstantPeriodicTask periodicWrapper = new InstantPeriodicTask(decoratedRun, executor);
            try {
                Future<?> f;
                if (initialDelay <= 0L) {
                    f = executor.submit(periodicWrapper);
                } else {
                    f = executor.schedule(periodicWrapper, initialDelay, unit);
                }
                periodicWrapper.setFirst(f);
            } catch (RejectedExecutionException ex) {
                RxJavaPlugins.onError(ex);
                return EmptyDisposable.INSTANCE;
            }

            return periodicWrapper;
        }
        ScheduledDirectPeriodicTask task = new ScheduledDirectPeriodicTask(decoratedRun);
        try {
            Future<?> f = executor.scheduleAtFixedRate(task, initialDelay, period, unit);
            task.setFuture(f);
            return task;
        } catch (RejectedExecutionException ex) {
            RxJavaPlugins.onError(ex);
            return EmptyDisposable.INSTANCE;
        }
    }

    /**
     * Wraps the given runnable into a ScheduledRunnable and schedules it
     * on the underlying ScheduledExecutorService.
     * <p>If the schedule has been rejected, the ScheduledRunnable.wasScheduled will return
     * false.
     * @param run the runnable instance
     * @param delayTime the time to delay the execution
     * @param unit the time unit
     * @param parent the optional tracker parent to add the created ScheduledRunnable instance to before it gets scheduled
     * @return the ScheduledRunnable instance
     */
    @NonNull
    public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);

        if (parent != null) {
            if (!parent.add(sr)) {
                return sr;
            }
        }

        Future<?> f;
        try {
            if (delayTime <= 0) {
                f = executor.submit((Callable<Object>)sr);
            } else {
                f = executor.schedule((Callable<Object>)sr, delayTime, unit);
            }
            sr.setFuture(f);
        } catch (RejectedExecutionException ex) {
            if (parent != null) {
                parent.remove(sr);
            }
            RxJavaPlugins.onError(ex);
        }

        return sr;
    }

    @Override
    public void dispose() {
        if (!disposed) {
            disposed = true;
            executor.shutdownNow();
        }
    }

    /**
     * Shuts down the underlying executor in a non-interrupting fashion.
     */
    public void shutdown() {
        if (!disposed) {
            disposed = true;
            executor.shutdown();
        }
    }

    @Override
    public boolean isDisposed() {
        return disposed;
    }
}
```

在NewThreadWorker的构造器中，创建了一个线程池，并且后续任务都交给这个线程池进行处理：

```java
public static ScheduledExecutorService create(ThreadFactory factory) {
    final ScheduledExecutorService exec = Executors.newScheduledThreadPool(1, factory);
    tryPutIntoPool(PURGE_ENABLED, exec);
    return exec;
}
```

可以看出，NewThreadWorker中创建了一个核心线程数为1 的ScheduledThreadPool。

再回到createWorker()中，其中调用了pool.get().getEventLoop()，那么我们看看对应的实现：

```java
public Worker createWorker() {
    return new EventLoopWorker(pool.get().getEventLoop());
}

public PoolWorker getEventLoop() {
    int c = cores;
    if (c == 0) {
        return SHUTDOWN_WORKER;
    }
    // simple round robin, improvements to come
    return eventLoops[(int)(n++ % c)];
}
```

可以看出这里就是从FixedSchedulerPool的Worker缓存池中取一个PoolWorker出来，然后用它构造一个EventLoopWorker。可能到这一步就会有人有疑问了，为什么这里要再把PoolWorker封装成EventLoopWorker传出去，而不直接传递PoolWorker？

这里就要介绍一下Scheduler、Worker和具体Task的关系了:

![image-20200625165824348](.\imgs\image-20200625165824348.png)

如上图所示，一个Scheduler可以创建多个Worker，每个Worker中可以执行多个Task。同一个 Worker 创建的 Task 都会确保串行，且立即执行的任务符合先进先出原则。Worker 绑定了调用了他的方法的 Runnable，当该 Worker 取消时，基于他的 Task 均被取消。

因此一般情形下，一个Worker中会有Task队列，用户可能会同时拥有多个Worker实例，当取消一个Worker时，不能够把其他Worker上的任务取消。

如果直接将FixedSchedulerPool中返回的PoolWorker提供给用户，由于PoolWorker是从缓存数组中获取的，因此用户可能会取到同一个PoolWorker，这会导致将这个Worker上的任务全部取消。

除此之外，当用户取消了PoolWorker时，其对应的线程也被释放，后面再次获取到该PoolWorker时就无法在其上运行任务了。

所以这里将PoolWorker封装成EventLoopWorker返回给用户，这样当用户取消EventLoopWorker时，并不会影响内部的PoolWorker。

相信到这里也差不多明白了ComputationScheduler的工作原理了，主要就是通过FixedSchedulerPool创建一个和cpu核心数相同的Worker缓存池，每个Worker被创建时都会创建自己的线程池，当用户调用createScheduler()获取Worker时，从FixedSchedulerPool中取出一个PoolWorker，将其封装给EventLoopWorker供用户使用。

#### IOScheduler

IOScheduler的实现就不详细分析了，这里就讲讲它的大致原理吧：

由于IO设备的速度远低于CPU速度，会发生阻塞，而在等待IO时，CPU往往是闲置的，因此英爱尽可能多的创建线程来提高CPU的利用率，但也不是越高越好，线程数目膨胀到一定程度既会影响 CPU 的效率，也会消耗大量的内存。因此需要有一个机制来清理闲置线程，在IOScheduler中，他是通过超时清除来控制线程数目的。 

