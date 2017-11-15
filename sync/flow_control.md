# Flow Control

Thread之间除了共用数据以外，流程同步也是常用到的一个技巧。如果把人当做一个thread，彼此之间协同合作就是flow control。是不是常常会有某A做完了一个动作之后，某B甚至某C才可以继续做后续的动作呢? 在multi-thread中也常常会有这样的需求。

## Wait and notify

在java中有关流程控制有一个最基本的primitive，那就是`Object#wait()`跟`Object#notify()`。当一个thread对一个对象调用`wait()`之后会完全卡住，要一直到另外一个thread触发`notify()`才可以继续往下执行。几乎所有有关流程控制的逻辑最底层都是透过wait跟notify来实现。

我打算用一个简化版的[Producer Consumer Pattern](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem)来解释flow control。通常producer负责产生message，而consuemer负责接收并处理message。中间会有一个queue，当queue是满的时produce，或queue是空的时consume，那caller thread就会被blocked，直到状态解除为止。但是为了方便解释起见，这边拿掉了queue，而只单纯的让producerd丢一个message给consumer而已。下面是一个例子:

```java
public class FlowControl {

    // Lock for synchronization
    private Object lock = new Object();
    // Message to pass
    private String message = null;

    public void produce(String message) {
        System.out.println("produce message: " + message);
        synchronized (lock) {
            this.message = message;
            lock.notify();
        }
    }

    public void consume() throws InterruptedException {
        System.out.println("wait for message");
        synchronized (lock) {
            lock.wait();
            System.out.println("consume message: " + message);
        }
    }


    public static void main(String[] args) throws InterruptedException {
        final FlowControl flowControl = new FlowControl();

        // Create consumer thread
        new Thread(()->{
            try {
                flowControl.consume();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        // Sleep for 1 sec
        Thread.sleep(1000);

        // Create producer thread
        new Thread(()->{
            flowControl.produce("helloworld");
        }).start();
    }
}
```

我们分别定义了`produce()`跟`consume()`两个methods。在consume中，我们会调用`lock.wait()`，注意wait这个method一定要让被调用的那个object被包在**sychronized block**之中，不然直接会有compile error。相对的，在produce中我们会呼叫`lock.notify()`，同样的也是要包在synchronized中。

在这个例子里面，我们也可以用`this`来取代`lock`物件。也就是用`sychronized(this)`, `this.wait()`, `this.notify()`取代。但这边会用一个独立的lock有个优点就是有比较好的隔离效果，以避免外部取得`FlowContorl`此class的instance的人，有机会影响内部结果。

最后看到`main` method，我们起了两个threads，先是consumer thread，他会等著consume一个message；再来睡了一秒钟后，我们产生了producer thread，他会produce一个`helloworld` message。最后跑出的结果就会像这样

```
wait for message
produce message: helloworld
consume message: helloworld
```

> 注意，`wait()`和`notify()`方法调用之前必须拿到object的锁，详情参见java api。

另外有一个`Object#notifyAll()` method其实跟`Object#notify()`类似前者一次叫醒所有waiting threads，后者只会叫醒一个waiting thread。

## Thread Join

另外一种更简单的做法就是thread join。**Join**这个动词在流程同步的时候常常会使用，它代表的是等待别人工作的完成；而相对的词是**Fork**，代表的是把工作发包给别的thread开始做。

下面的代码则是一个简单的范例，我们产生一个worker thread来执行我们的task，在我们的main thread中去**join**这个worker

```java
// Create workder thread
Thread worker = new Thread(() -> {
    try {
        System.out.println("worker start");
        Thread.sleep(1000);
        System.out.println("worker complete");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});

worker.start();

System.out.println("master wait");
worker.join();
System.out.println("master complete");
```

此代码的output会像下面这样，可以看到master要等worker完成后，才会继续往下执行。

```
master wait
worker start
worker complete
master complete
```

## InterruptedException

在如上例子中总是出现InterruptedException异常，这个异常很烦，什么时候要捕获？为什么要捕获这个异常？
1. sleep
2. wait
3. join

我们发现每次线程再调用阻塞线程的方法的时候，会强制捕获这个异常，因为如果一个线程是不可打断的，那么当他阻塞的时候就只能等待他结束了，如果程序设计不合理，这个线程就有可能永远不会停止，所以默认jdk提供的阻塞方法总是会使线程进入可被打断的阻塞状态，这样，别的线程可以随时调用某线程对象的`interrupt()`方法打断线程，这时阻塞线程就会恢复运行状态，进入`catch(InterruptedException)`流程，这样，就强制每个线程进入阻塞状态时为他安装被打断处理。

## Other Utilities

碍于篇幅关系(~~还是因为懒~~)，其他还有一些工具可以帮你做flow control

- [CyclicBarrier](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/CyclicBarrier.html): Barrier算是一个流程同步的时常用的术语，代表的是所有的threads都需要到达某著stage的时候，才继续往下走。这概念就很像出游的时候，大家会在一个地点集合，全员到齐了再一起出发的概念。
- [CountDownLatch](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/CountDownLatch.html): 其实就如其名，是一个倒数的概念。每呼叫一次`countdown()`则倒数的counter就会减一，当一个thread去呼叫`await()`时，会一直卡在那边，直到counter归零为止，才会继续往下走。
- [Future](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/Future.html) and [Completable Future](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/CompletableFuture.html): 这个是Flow Control更高层次的封装，因为内容比较多，我留在后面的章节再做讨论。
