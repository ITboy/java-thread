#Future

[Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)是代表一个非同步呼叫的回传结果，而这个结果会在未来某一个时间点可以取得。这洋讲有点抽象，那举个例子好了，送洗衣服就是一个非同步的呼叫，因为衣服是交给别人洗而非自己洗，而洗好的衣服是一个**未来**会发生的结果，这个结果被Future这个class包装起来。

我们再更具体一点的把它变成实际的程式码，`Future`是一有一个型态惨数的class，使用上大概会像这洋

```java
Future<Clothes> future = laundryService.serviceAsync(myClothes);
```

洗衣店提供了一个非同步的服务，所以回传一个`Future`代表的是非同步的结果(Asynchronous Result)。因为是非同步，所以这个method不会卡住。拿到这个future之后，我们可以选择做别的事情，或是马上blocking取得。

```java
// block until result is available
Clothes clothes = future.get();
```

因为是非同步，所以我们可以一次执行很多个非同步的task，让他们可以平行的去处理，最后再一次等所有的非同步结果。这在生活中也挺常发生，例如我去夜市买鸡排豆花跟珍奶，我也不会呆呆的在每一个摊位前面等他一个一个做好，我可能会跟老板说等等过来拿，让这几个摊位可以**平行**的处理我的tasks。

Future是一个interface，所以需要具体的实作。但通常不需要自己实作，还记得前面[ThreadPool](basicpool.md)的章节就有提到`Future`了吗? 事实上[ExecutorService#submit](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html#submit-java.util.concurrent.Callable-)就提供了一个非同步执行的实作，并且回传一个`Future`，一般来讲我们只要使用这个method来实作我们的非同步执行就可以了。

所以上面的程式码的具体实作可能长这洋

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

如此一来，这个LaundaryService当中有一个threadpool帮忙洗衣服，当呼叫`serviceAsync`，则会产生一个task在threadpool中执行，并且先回传一个Future代表这个非同步的结果，就很像是一个取货单一洋。caller在之后再透过future取得最后完成的结果。