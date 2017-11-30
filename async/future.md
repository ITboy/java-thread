#Future

[Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)是代表一个非同步呼叫的回传结果，而这个结果会在未来某一个时间点可以取得。这洋讲有点抽象，那举个例子好了，送洗衣服就是一个非同步的呼叫，因为衣服是交给别人洗而非自己洗，而洗好的衣服是一个**未来**会发生的结果，这个结果被Future这个class包装起来。

我们再更具体一点的把它变成实际的程式码，`Future`是一有一个形参的class，使用上大概会像这洋

```java
Future<Clothes> future = laundryService.serviceAsync(myClothes);
```

洗衣店提供了一个非同步的服务，所以回传一个`Future`代表的是非同步的结果(Asynchronous Result)。因为是非同步，所以这个method不会卡住。拿到这个future之后，我们可以选择做别的事情，或是马上blocking取得。

```java
// block until result is available
Clothes clothes = future.get();
```

因为是非同步，所以我们可以一次执行很多个非同步的task，让他们可以平行的去处理，最后再一次等所有的非同步结果。这在生活中也挺常发生，例如我去夜市买鸡排豆花跟珍奶，我也不会呆呆的在每一个摊位前面等他一个一个做好，我可能会跟老板说等等过来拿，让这几个摊位可以**平行**的处理我的tasks。

Future是一个interface，所以需要具体的实现。但通常不需要自己实现，还记得前面[ThreadPool](basicpool.md)的章节就有提到`Future`了吗? 事实上[ExecutorService#submit](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html#submit-java.util.concurrent.Callable-)就提供了一个非同步执行的实现，并且回传一个`Future`，一般来讲我们只要使用这个method来实现我们的非同步执行就可以了。

所以上面的程式码的具体实现可能长这样

```java
public class LaundaryService {
    private ExecutorService executorService = /*...*/;

    public Future<Clothes> serviceAsync(Clothes dirtyClothes) {
        return executorService.submit(()-> service(dirtyClothes));
    }

    public Clothes service(Clothes dirtyClothes) {
        return dirtyClothes.wash();
    }
}
```

如此一来，这个LaundaryService当中有一个threadpool帮忙洗衣服，当呼叫`serviceAsync`，则会产生一个task在threadpool中执行，并且先返回一个Future代表这个非同步的结果，就很像是一个取货单一样。caller在之后再透过future取得最后完成的结果。

下面具体看一下`Future`的官方定义和提供的方法：
`Future`是异步执行的顺时结果，他保存异步任务的执行结果和执行状态，提供以下方法：
1. `V get()` - 取得异步任务的值，可能阻塞当前线程，知道异步任务执行结束，拿到执行结果返回。
2. `isCanceled()` - 任务是否已经被取消，如果还未结束或已经执行成功，就返回false，否则返回true
3. `isDone()` - 任务是否已经结束，不代表任务一定成功，如果任务被取消，仍然返回true，该值标明任务是仍然在执行还是已经结束。
4. `cancel(boolean mayInterruptIfRunning)` - 取消任务执行，如果任务已经执行结束，返回false表明取消失败，如果任务还没有执行结束，取消有两种情况，一种是任务还在队列中，没有开始执行，此时直接将任务从队列移除，另一种是当前任务正在被执行，参数`mayInterruptIfRunning`为`true`表示此时要打断正在执行任务的线程，强制取消，如果`mayInterruptIfRunning`设为`false`则等待当前任务执行结束。

所以`Future`接口封装了一系列对异步执行结果进行操作的基本方法，包括取得执行结果，取得当前状态，取消任务。`Future`使得我们可以先把一个耗时长的任务丢给线程池处理，而自己接下去做别的事情，等到想去取之前任务的执行结果时再去取，但是仍然有限制，就是去取结果仍然是一个同步的过程，除非任务已经执行结束，否则会阻塞当前线程，直到取到结果，而无法在任务执行结束时得到通知或调用提前安装的回调函数，而jdk1.8中的`CompletableFuture`为这些问题提供了解决方案，可以参看下一节内容。
