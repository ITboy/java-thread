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

### 产生`CompletableFuture`

#### 构造函数

构造函数构造一个未完成状态的`CompletableFuture`，需要显式调用`CompletableFuture#complete`等来触发他的一系列回调函数。

```java
		CompletableFuture<Integer> cf = new CompletableFuture<Integer>();
		cf.thenAccept((i) -> {
			System.out.println(i);
		});
		cf.complete(1024); // 没有这一行，就不会打印任何信息
```

#### 工厂函数

* `static <U> CompletableFuture<U>	completedFuture(U value)`
    等同于`Promise.resolve(value)`，返回一个已经以value完成的`CompletableFuture`
    上面例子等价于：
```java
    CompletableFuture.completedFuture(1024).thenAccept((i) -> {
    	System.out.println(i);
    });
```

* `static CompletableFuture<Void>	runAsync(Runnable runnable)`
    返回一个`CompletableFuture`，当`runable`执行完成，这个`CompletableFuture`也完成。
    runnable在`ForkJoinPool.commonPool()`中被执行。

* `static CompletableFuture<Void>	runAsync(Runnable runnable, Executor executor)`
    同上，可以指定`executor`。

* `static <U> CompletableFuture<U>	supplyAsync(Supplier<U> supplier)`
    通`runAsync`，可以提供返回值。

* `static <U> CompletableFuture<U>	supplyAsync(Supplier<U> supplier, Executor executor)`
    同上，可以指定`executor`。

### 合并多个`CompletableFuture`

* `static CompletableFuture<Void>	allOf(CompletableFuture<?>... cfs)`
* `static CompletableFuture<Object>	anyOf(CompletableFuture<?>... cfs)`

```java
	public static void main(String[] args) throws InterruptedException, ExecutionException {
		CompletableFuture.allOf(delayValue(100, 2), delayValue(200, 5)).thenRun(() -> {
			System.out.println("all is finished");
		});
		CompletableFuture.anyOf(delayValue(100, 2), delayValue(200, 5)).thenRun(() -> {
			System.out.println("any is finished");
		});
		Thread.sleep(1000*5);
	}
	
	private static CompletableFuture<Integer> delayValue(int i, int seconds) {
		return CompletableFuture.supplyAsync(() -> {
			try {
				Thread.sleep(seconds * 1000);
			} catch (Exception e) {
				e.printStackTrace();
				return 0;
			}
			return i;
		});
	}
```
Console:
```
any is finished
all is finished
```
> 注意：这里`CompletableFuture`与`Promise`不同，需要在最后调用`Thread#sleep()`或`Future#get()`，推测可能因为在`CompletableFuture`内部的线程被表示为`daemon`，当主线程结束后，由于没有非`daemon`线程，进程强制结束。

### 常用函数
1. `thenApply()`,`thenRun()`, `thenAccept`类似于`Promise#then()`，分别对应与处理函数的原型，是否有参数，是否有返回值，注意这些函数都是当`CompletableFuture`完成时才会调用，而出现异常时并不会调用。
2. `thenCompose()`对应`Promise#then()`的回调函数返回一个Promise的情况，他要求处理函数返回一个`CompletableFuture`，可以认为处理函数返回的`CompletabledFuture`就成了`thenCompose()`返回的`CompletableFuture`。
3. `whenComplete()`, `whenCompleteAsync()`有点像`finally`关键字，不论`CompletableFuture`执行结果如何都会走到他的回调函数，complete的值或异常值会以参数传递给回调函数。
>这里的回调函数只能是`Comsumer`，不能有返回值，但是`whenComplete()`返回的`CompltetableFuture`总是以当前`CompletableFuture`完成的值作为完成值，除非回调函数出现异常，这时，返回的`CompletableFuture`以回调函数抛出的异常作为返回`CompletableFuture`的异常值。
4. `handle()`, `handleAsync()`跟`whenComplete()`系列类似，都类似`finally`，区别有两点：
    * handle接收的是一个`BiFunction`，有返回值，而`whenComplete`接收一个`Comsume()`，没返回值
    * 当回调函数正常结束时，`handle()`以回调函数的返回值为返回的`CompletableFuture`的完成值，而`whenComplete()`会预留当前`CompletableFuture`的完成值作为返回`CompletableFuture`的完成值。
> 从名字和功能上区分`whenComplete()`和`handle()`的使用区别：
> `whenComplete()`认为不会修改当前`CompletableFuture`的完成值和异常值，虽然可以通过在回调函数中抛出异常来覆盖当前`CompletableFuture`的异常值，但是通常意义上，`whenComplete()`意思是当`CompletableFuture`结束时的后续处理，根据complete值或异常值加一些额外的处理，不修改状态和值，返回的`CompletableFuture`的状态和当前`CompletableFuture`状态一致。
> 而`handle()`被用作一个新的转换，得到一个全新的`CompletableFuture`，根据当前`CompletableFuture`的结果处理得到一个全新的`CompletableFuture`。
5. `exceptionally`
等同于`Promise#catch()`。

```java
	public static void main(String[] args) throws InterruptedException, ExecutionException {
		exceptionFuture().whenComplete((Void value, Throwable t) -> {
			System.out.println(t.getMessage()); // don't change exception
		}).thenAccept((Void value) -> {
			System.out.println("reserved last completed value: " + value);
		}).exceptionally((t) -> {
			System.out.println(t.getMessage()); // exception will be delivery to here
			return null;
		});
		Thread.sleep(1000*5);
	}
	
	private static CompletableFuture<Void> exceptionFuture() {
		return CompletableFuture.supplyAsync(() -> { throw new RuntimeException(); });
	}
```

### 限制
1. 只能抛出非Checked的Exception
2. 链式调用后面产生的`CompletableFuture`受前面的限制，不能随心所欲。
`exceptionally()`作为一个根据`Throwable`产生一个全新的`CompletableFuture`，但这个`CompletableFuture`必须完成一个某种类型的值，这个类型的值必须跟调用`exceptionally()`的`CompletableFuture`完成的值严格一致。

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