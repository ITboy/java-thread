#Basic Pool

对比[thread](thread.md)中的线程使用的五个场景，其实不外乎是想使用一个独立的线程去执行一个任务，直接使用线程的方法可以用`Runnable`接口描述一个任务，`new Thread`来执行这个任务。相比这种办法，线程池是一种新的执行任务的方式，本质上是使得任务和任务如何被执行解耦。
使用线程：
```java
new Thread(new Runnable() {
    ...
}).start();
```

使用线程池：
``` java
Executor.execute(new Runnable() {
});
```

所以大多数情况下，我们无需在直接使用线程，而**直接使用线程池**来执行任务。

## `Excutor`

`Executor`接口只有一个execute方法，他表示自己是一个执行者，可以执行`Runnable`接口的实现，本身是跟线程池无关的概念。

从功能上来看Thread其实也是一个执行者，他也要求传递一个`Runnable`的实现，把这个`Runnable`放在一个线程中执行，因此他是一个特殊的执行者，因为执行者的功能只有执行任务这个含义，不要求是在当前线程还是另辟线程，还是交由线程池，还是有多少延迟后在执行，仅仅执行。

## `Runnable`与`Callable`

这两个接口作为基础接口在后面会经常使用，都可以表示被执行的任务，区别在于：
1. `Runnable`接口的函数不可以有返回值，`Callable`接口的函数有返回值，并且使用泛型来传入返回值类型。
2. `Runnable`接口的函数不能抛出checked异常，而`Callable`可以。

```java
Interface Callable<V> {
    V call() throws Exception;
}
```

```java
Interface Runnable {
    void run();
}
```

## ExcutorService

在Java中，thread pool都会实现一个接口[Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html)，事实上更明确的说是实现[ExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html)这个接口。前者只定义了一个简单的`execute` method，就跟我前面一个章节的`execute`定义一模一样，就是在thread pool中执行一个task。

```java
public interface Executor {
    void execute(Runnable command);
}
```

后者继承了Exectuor接口，定义了更多的method，

```java
public interface ExecutorService extends Executor {
    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

Methods | Description
--------|-----------------
submit | 可以调用没有回传值的task(Runnable)跟有回传值的task(Callable)，并且返回一个[Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)。这个Future的概念容我稍后介绍。
invokeAll| 一次执行多个task，并且取得所有task的future objects 
invokeAny | 一次执行多个task，并且取得第一个完成的task的future object
shutdown<br>shutdownNow<br> | 让整个thread pool的threads都停止，简单讲就是打烊了。
awaitTermination | 等待所有shutdown后的task都执行完成。可以说是打烊并且所有善后都处理完了。

另外还有一种较特殊的thread pool称为[ScheduledExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html)。他除了可以做原本submit task到thread pool以外，还可以让这个task不会立刻执行。他的定义如下:

```java
public interface ScheduledExecutorService extends ExecutorService {

    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);

    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);

    
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

    
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);

}
```

Methods | Description
--------|-----------------
schedule | 让task可以在一段时间之后才执行。
scheduleAtFixedRate | 除了可以让task在一段时间之后才执行之外，还可以固定周期执行

在看完thread pool的抽象定义之后，我们来看看有哪些现成的实例可以拿来使用。

## Executors

在Java中，大部分的thread pool都是透过[Executors](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html)来产生，里面有非常多的factory method来去产生不同类型的thread pool。

Name | Description
-----|---------------
Executors.newSingleThreadPool() <br> Executors.newSingleThreadScheduledExecutor() | 产生一个只有一个thread的thread pool。
Executors.newFixedThreadPool(int nThreads) | 产生固定thread数量的thread pool
Executors.newCachedThreadPool() | 产生没有数量上限的thread pool。Thread的数量会根据使用状况动态的增加或是减少。
Executors.newScheduledThreadPool(int corePoolSize) | 产生固定数量可以做scheduling的thread pool。
Executors.newWorkStealingPool() | 产生work stealing的thread pool，这种类型的thread pool下一章会介绍。


基本上大部分的情况之下，我们已经不会自己产生thread了，透过thread pool的方式可以更有效的管理我们的threads。再想想前面的银行取票机的例子，也许一个银行需要一般业务的取票机，另外有一个外汇的取票机，还有可能一个专属于VIP的取票机。根据不同的业务需求，我们用不同的thread pool去管理。可以避免较为重要的工作，反而被一些比较不重要但是做比较久的task卡住。在我们的multi thread的环境之下，我们也会有类似的情境。

接下来我们介绍一个比较特别的pool，称之为ForkJoinPool。