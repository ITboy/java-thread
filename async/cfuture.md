#CompletableFuture

Java8在语言层级推出了lambda之后，也随着推出了支持lambda的函数库。其中我把[Stream API](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html), [Optional API](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html), 跟[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)称为Java8三神器。这三个都有Functional Language裡[Monad](https://en.wikipedia.org/wiki/Monad_(functional_programming%29)的精神，而CompletableFuture也就是Monadic Future。这边我还是先不要讨论太多Functional Language，让我们来直接看看CompletableFuture怎么使用。

先来看一个最简单的例子
```java
CompletableFuture<Void> future =
CompletableFuture.runAsync(() -> {
    try {
        Thread.sleep(1000);
        System.out.println("hello");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});

future.get();
System.out.println("world");
```
以上的代码会印出
```
hello
world
```

透过lambda的特性，我们同样可以把非同步调用弄的跟开thread一样简单。在CompletableFuture定义了几个static methods，帮助我们快速的非同步执行。

Method | Description
-------|-------------------
[runAsync(Runnable runnable)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#runAsync-java.lang.Runnable-) | 非同步的执行一个没有回传值的task，并且在预设的thread pool中执行。预设为 [ForkJoinPool.commonPool()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--)
[runAsync(Runnable runnable, Executor executor)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#runAsync-java.lang.Runnable-java.util.concurrent.Executor-) | 非同步的执行一个没有回传值的task，并且在指定的thread pool之中执行。
[supplyAsync(Supplier<U> supplier)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-) | 非同步的执行一个有回传值的task，并且在预设的thread pool之中执行。
[supplyAsync(Supplier<U> supplier, Executor executor)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-java.util.concurrent.Executor-) | 非同步的执行一个有回传值的task，并且在指定的thread pool之中执行。

咦? 那所谓的Completable是什麽意思? 那前一章所介绍的Future有什麽不一样? 事实上CompletableFuture是一个Future的实例，至于Completable，我打算以这四个特性来讨论

1. Completable
2. Listenable
3. Composible
4. Combinable

## Completable

所谓的Completable就是这个future可以被complete。其实这要先讨论Future跟Promise这两个概念。

1. Future: 是一个未来会完成的一个结果，算是这个结果的容器。Caller透过Future来等非同步执行的结果。
2. Promise: 是可以被改变可以被完成的值，通常是非同步执行的结果。Callee透过Promise来告知非同步完成的结果。

基本上就是一体两面啦。对于asynchronous invocation，对于caller看到就是future，对于callee就是看到promise。而CompletableFuture就同时扮演了Future跟Promise两种角色。

所以CompletableFuture会被下面这样使用

1. 在非同步调用时，会先产生一个CompletableFuture，并且回传给caller
2. 这个CompletableFuture会连同async task一起传到worker thread中。
3. 当执行完这个async task，callee会呼叫CompletableFuture的`complete()`
4. 此时caller可以透过CompletableFuture的`get()`取得结果的值。

其实这跟我们在[Flow Control](flow_control.md)的章节看到的`wait()`/`notify()`极为相似，比较不一样的就是这不只是流程同步，还带有回传值。除了complete以外，当执行错误的时候，也可以调用`completeExceptionally()`。

在completable这个特性里，我们把属于caller/consumer用的Future介面，以及callee/provider用的Completable放在一起，我们来检视一下有哪些跟Completable相关


Method |  Description
-------|----------------------
[complete(T t)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#complete-T-) | 完成非同步执行，并回传结果
[completeExceptionally(Throwable ex)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#completeExceptionally-java.lang.Throwable-) | 非同步执行不正常的结束

有了以上的概念，我们很快的可以很快地写出`CompletableFuture.runAsync()`可能的逻辑

```java
public static CompletableFuture<Void> runAsync(Runnable runnable) {
    CompletableFuture<Void> future = new CompletableFuture<>();
    ForkJoinPool.commonPool().execute(() -> {
        try {
            runnable.run();
            future.complete(null);
        } catch (Throwable throwable) {
            future.completeExceptionally(throwable);
        }
    });

    return future;
}
```

在Google的[Guava library](https://github.com/google/guava)中也可以看到completable的踪影，那就是[SettableFuture](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/SettableFuture.html)。

## Listenable

对于asynchronous invocation的caller来讲，`Future`只提供了一个pulling result的方法，更多时候我们想要的是**好了叫我**这种语意。因此*Listenable*的特性，就是我们可以注册一个callback，让我可以listen执行完成的event。

在CompletableFuture主要是透过[whenComplete()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#whenComplete-java.util.function.BiConsumer-)跟[handle()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#handle-java.util.function.BiFunction-)这两个method。

Method |  Description
-------|----------------------
[whenComplete()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#whenComplete-java.util.function.BiConsumer-) | 当完成时，把result或exception带到callback function中。
[handle()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#handle-java.util.function.BiFunction-) | 当完成时，把result或exception带到callback function中，并且回传最后的结果。

我再把最上面的例子改写成用listener的方式
```java
CompletableFuture.runAsync(() -> {
    try {
        Thread.sleep(1000);
        System.out.println("hello");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}).whenComplete((result, throwable) -> {
    System.out.println("world");
});
```

这两个method以及包含后面会提到的method都有三种变形，分别是

- xxxx(function): function会用前个执行的thread去调用。
- xxxxAsync(function): function会用非同步的方式调用，并用预设的thread pool。
- xxxxAsync(function, executor): function会用非同步的方式调用，并用指定的thread pool。

由于基本逻辑相似，之后就不再重述。

同样在Guava library中也可以看到listenable的踪影，那就是[ListenableFuture](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFuture.html)。

## Composible

有了Listenable的特性之后，我们就可以做到当完成时，在做下一件事情。如果接下来又是一个非同步的工作，那就可能会串成非常多层，我们称之为callback hell。下面是个例子


```java
public static void sleep(long time) {
    try {
        System.out.printf("sleep for %d milli\n", time);
        Thread.sleep(time);
        System.out.printf("wake up\n");
    } catch (InterruptedException e) {
    }
}

public static void main(String[] args) throws InterruptedException {
    CompletableFuture<Void> future = 
    CompletableFuture
    .runAsync(() -> sleep(1000))
    .whenComplete((result, throwable) -> {
        if (throwable != null) {
            return;
        }

        CompletableFuture
        .runAsync(() -> sleep(1000))
        .whenComplete((result2, throwable2) -> {
            if (throwable2 != null) {
                return;
            }

            CompletableFuture
            .runAsync(() -> sleep(1000))
            .whenComplete((result3, throwable3) -> {
                if (throwable2 != null) {
                    return;
                }

                System.out.println("Done");
            });
        });
    });
```

这个程式码这样三层可能已经受不了了，如果更多层应该会有噁心的感觉。这还不打紧，如果再加上错误处理，那可能更是晕头转向。

对于这种一连串的invocation，如果可以把这些async function组起来，变成一个单一future，可能会舒服许多。先来看最后的结果，我们再来讨论细节。

```java
CompletableFuture
.runAsync(() -> sleep(1000))
.thenRunAsync(() -> sleep(1000))
.thenRunAsync(() -> sleep(1000))
.whenComplete((r, ex) -> System.out.println("done"));
```

有没有觉得清爽许多?这就是Composible的魅力。

在CompletableFuture中，它提供了非常多的compose methods来帮助我们组合各种sync methods变成async methods。我来列举一下

Method | Trasnformer | To Type
-------|-----------|-----------
[thenRun()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenRun-java.lang.Runnable-) | ```Runnable``` | ```CompletableFuture<Void>```
[thenAccept()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenAccept-java.util.function.Consumer-) | ```Consumer<T>``` | ```CompletableFuture<Void>```
[thenApply()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenApply-java.util.function.Function-) | ```Function<T, U>``` |  ```CompletableFuture<U>```
[thenCompose()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCompose-java.util.function.Function-) | ```Function<T, CompletableFuture<U>>``` |  ```CompletableFuture<U>```

型态的部分我有稍微调整一下，让它比较容易读。但是我们都可以看到他们都有一个特性，就是把原本某个CompletableFuture的type parameter，经过一个transformer后，转成另外一个Type的CompletableFuture，这就是Monad中的**map**。而最后一个因为他的回传值本来就是CompletableFuture，这种转换我们称之为**flatmap**。其实同样的概念在Optional API跟Stream API都找得到，有兴趣可以去寻宝一下。

这些method也都有`xxx()`, `xxxAsync(func)`, `xxxAsync(func, executor)`三个版本，就如前面所述。

经过这样的转换过程，我们把很多的future合併成单一的future。这些转换我们没有看到任何的exception处理，因为在任何一个阶段出现exception，对于整个包起来的future就是exception。所以我们就是希望把每一个小的async invocation **compose**成一个大的async invocation。

同样在guava library中，我们可以看到composible的踪影，他是放在[Futures](https://google.github.io/guava/releases/19.0/api/docs/com/google/common/util/concurrent/Futures.html)下面的`transformXXX()`相关的methods。

## Combinable

最后，async的流程有些时候不会是单一条路的，有时候更像是[DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)(Directed Acyclic Graph)。例如做一个爬虫程式(Crawler)，我们排一个文章的时候，可能会抓到很多个外部链结，这时候就会继续展开更多非同步的task。等到到了某个停止条件，我们就要等所有爬虫的task完成，最终等于执行完这个大的async task。

这时候我们会希望把多个future完成时当作一个future的complete，这就是combinable的概念。跟composible的概念不同的是，composible是一个串一个，比较像是串连的感觉；相对的combinable，就比较像是并联。

来看看CompletableFuture针对这种应用有哪些method，假设原始形态`CompletableFuture<T>`


Method | With | Transformer | Return Type
-------|-----------|-----------
[runAfterBoth()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#runAfterBoth-java.util.concurrent.CompletionStage-java.lang.Runnable-) | `CompletableFuture<?>` | `Runnable` | `CompletableFuture<Void>`
[runAfterEither()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#runAfterEither-java.util.concurrent.CompletionStage-java.lang.Runnable-) | `CompletableFuture<?>` | `Runnable` | `CompletableFuture<Void>`
[thenAcceptBoth()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenAcceptBoth-java.util.concurrent.CompletionStage-java.util.function.BiConsumer-) | `CompletableFuture<U>` | `BiConusmer<T,U>` | `CompletableFuture<Void>`
[acceptEither()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#acceptEither-java.util.concurrent.CompletionStage-java.util.function.Consumer-) | `CompletableFuture<T>` | `Conusmer<T>` | `CompletableFuture<Void>`
[applyToEither()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#applyToEither-java.util.concurrent.CompletionStage-java.util.function.Function-) | `CompletableFuture<T>` | `Function<T,U>` | `CompletableFuture<U>`
[thenCombine()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCombine-java.util.concurrent.CompletionStage-java.util.function.BiFunction-) | `CompletableFuture<U>` | `BiFunction<T,U,V>` | `CompletableFuture<V>`

跟Composible那边的method不一样的是多了一个*with*，代表的是combine的对象。这些method都有可以把两个future **combine**成一个future的特色。而**both**跟**either**，代表的是两个都完成才算完成，还是其中一个完成则算完成。

除了两两combine的这些method以外，CompletableFuture还有提供两个static methods来做combine多个futures。

Method | Description
-------|-------------
[allOf(...)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#allOf-java.util.concurrent.CompletableFuture...-) | 回传一个future，其中所有的future都完成此future才算完成。 
[anyOf(...)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#anyOf-java.util.concurrent.CompletableFuture...-) | 回传一个future，其中任何一个future完成则此future就算完成。

## `CompletableFuture` API

`CompletableFuture`实现了`Future`和`CompletionStage`两个接口。这样`CompletableFuture`就代表了一个异步执行的结果以及执行的时期，可以为他的执行结束添加回调函数。
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

## 总结

CompletableFuture跟lambda的组合，在java8中带来了非同步的生力军。Lambda让之前的annoymous inner class来实作async task会变成简洁非常多，而Completable future又多了composible跟combinable，让复杂的非同步流程变得非常的简洁。

再来就如前面讲的，大部分的method都有**async**，以及**async with executor**的版本。所以我们可以很明确指定到底我的task是摆在哪一个thread pool跑。对于UI程式，常常有一个pattern就是先async到worker thread pool去执行，处理完再到UI thread去update UI并且呈现，这个流程在新的CompletableFuture下变得更为简洁容易。

