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

## `Future`

`Future`是异步执行的顺时结果，他保存异步任务的执行结果和执行状态，提供以下方法：
1. `V get()` - 取得异步任务的值，可能阻塞当前线程，知道异步任务执行结束，拿到执行结果返回。
2. `isCanceled()` - 任务是否已经被取消，如果还未结束或已经执行成功，就返回false，否则返回true
3. `isDone()` - 任务是否已经结束，不代表任务一定成功，如果任务被取消，仍然返回true，该值标明任务是仍然在执行还是已经结束。
4. `cancel(boolean mayInterruptIfRunning)` - 取消任务执行，如果任务已经执行结束，返回false表明取消失败，如果任务还没有执行结束，取消有两种情况，一种是任务还在队列中，没有开始执行，此时直接将任务从队列移除，另一种是当前任务正在被执行，参数`mayInterruptIfRunning`为`true`表示此时要打断正在执行任务的线程，强制取消，如果`mayInterruptIfRunning`设为`false`则等待当前任务执行结束。

所以`Future`接口封装了一系列对异步执行结果进行操作的基本方法，包括取得执行结果，取得当前状态，取消任务。

## `CompletionStage`

`Future`有一个很大的缺点，作为一个异步执行的结果，他只能使用阻塞的方式来主动获取结果，或者不断的循环来查看执行的状态，不能在异步执行结束时被通知，所以在读取执行结果仍然要使用同步的方式。

CompletionStage是一个接口，允许为异步执行添加回调函数。根据API的描述：

> A stage of a possibly asynchronous computation, that performs an action or computes a value when another CompletionStage completes. A stage completes upon termination of its computation, but this may in turn trigger other dependent stages.

`CompletionStage`执行一个动作或计算一个值的时期，他是另一个`CompletionStage`完成时触发，当他执行结束后也会触发另一个依赖的`CompletionStage`。

关于接口的更多信息，参考 [the-future-is-completable-in-java-8/](http://www.jesperdj.com/2015/09/26/the-future-is-completable-in-java-8/)和官方API。

## `CompletableFuture`

实现了`Future`和`CompletionStage`。这样`CompletableFuture`就代表了一个异步执行的结果以及执行的时期，可以为他的执行结束添加回调函数。
API手册的定义：

> A Future that may be explicitly completed (setting its value and status), and may be used as a CompletionStage, supporting dependent functions and actions that trigger upon its completion.

他是一个可以被显式完成的future，也就是可以设置他的执行结果和状态，并且可以被作为`CompletionStage`来使用，支持当他执行完成的时候触发依赖的function和动作。


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
scheduleAtFixedRate | 除了可以让task在一段时间之后才执行之外，还可以固定週期执行

在看完thread pool的抽象定义之后，我们来看看有哪些现成的实作可以拿来使用。

## Executors

在Java中，大部分的thread pool都是透过[Executors](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html)来产生，裡面有非常多的factory method来去产生不同类型的thread pool。

Name | Description
-----|---------------
Executors.newSingleThreadPool() <br> Executors.newSingleThreadScheduledExecutor() | 产生一个只有一个thread的thread pool。
Executors.newFixedThreadPool(int nThreads) | 产生固定thread数量的thread pool
Executors.newCachedThreadPool() | 产生没有数量上限的thread pool。Thread的数量会根据使用状况动态的增加或是减少。
Executors.newScheduledThreadPool(int corePoolSize) | 产生固定数量可以做scheduling的thread pool。
Executors.newWorkStealingPool() | 产生work stealing的thread pool，这种类型的thread pool下一章会介绍。


基本上大部分的情况之下，我们已经不会自己产生thread了，透过thread pool的方式可以更有效的管理我们的threads。再想想前面的银行取票机的例子，也许一个银行需要一般业务的取票机，另外有一个外汇的取票机，还有可能一个专属于VIP的取票机。根据不同的业务需求，我们用不同的thread pool去管理。可以避免较为重要的工作，反而被一些比较不重要但是做比较久的task卡住。在我们的multi thread的环境之下，我们也会有类似的情境。

接下来我们介绍一个比较特别的pool，称之为ForkJoinPool。