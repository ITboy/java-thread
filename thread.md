# 创建线程

在Java中Thread类的产生对象代表一个线程对象，一般而言创建线程都是使用Thread的几个构造函数，当然你也可以继承Thread实现自己的线程类。

```java
new Thread(() -> {
    System.out.println("hello thread");
}).start();
```

都已经java8的时代了，我直接用java8的语法来介绍java thread。在上面的程序中产生了一个新的thread，thread的constructor是一个实例`java.lang.Runnable`的对象。当调用`start()`此method时，则会启动这个thread，并且执行`Runnable#run()`。在java8中有好用的lambda，我们直接用lambda来实作那个runnable。当然你也可以用传统的产生一个新的class来实例`Runnable`。因为这个文件不是要教你怎么写java，我就先假设你已经了解写java的基本常识，如果你到这边还不懂我在写什么，那建议可以先拿一本java入门书来了解java最基本的编程知识。

# 线程的状态

线程有一下几种状态，使用Thread.State内部类表示：
1. NEW
A thread that has not yet started is in this state.
2. RUNNABLE
A thread executing in the Java virtual machine is in this state.
3. BLOCKED
A thread that is blocked waiting for a monitor lock is in this state.
4. WAITING
A thread that is waiting indefinitely for another thread to perform a particular action is in this state.
5. TIMED_WAITING
A thread that is waiting for another thread to perform an action for up to a specified waiting time is in this state.
6. TERMINATED
A thread that has exited is in this state.

# Demon线程

线程可以被标记为daemon，当main函数开始执行时，会启动一个jvm进程，如果所有的非daemon线程都死掉，那么jvm进程就会结束，daemon线程也会被迫结束。

所以，daemon线程应该是一个非重要的线程，他就像weak reference一样，进程不会因他的存在而不结束自己。

main线程默认被标记为非daemon线程。

# 线程的结束

线程什么时候会结束：
1. 当线程的代码自然执行到最后一行或抛出未处理的异常
2. 当进程结束时，所有线程都会结束
3. `Thread.stop()`已经被废弃，推荐在线程内部维护一个变量来决定线程是否结束

# 进程的结束

在jvm内部有两种引起进程结束的途径：
1. `Runtime.exit()`的调用
强制结束jvm进程，不论其他线程正在做什么
2. 所有非daemon线程结束

# 使用场景

通常有以下几种情况会用thread

1. IO相关的task，或称IO bound task。如果同时需要读很多个文件，或是同时要处理很多的sokcet connection，用thread的方法去做blocking read/write可以让程式不会因为等待IO而导致什么事情都不能做。
2. 执行很耗运算的task，或称CPU bound task。当这种task多，我们会想要使用多个CPU cores的能力。单线程的程序只能用到single core的好处，也就是程式再怎么耗CPU，最多就用到一个CPU。当使用multi-thraed的时候，就可以把CPU吃饱吃满不浪费。
3. 非同步执行。其实不管是IO-bound task或是CPU-bound task，我们应该都还是要跟主程式做沟通，所以通常的概念都是开一个thread去做IO bound或是CPU bound task，等到他们做完了，我再把结果拿到我的主程式做后续的处理。当然新开thread不是唯一的方法，我们稍后的章节会陆续提到很多非同步执行的策略跟方法。
4. 排程。常见的排程方法有三种。第一种是delay，例如一秒钟后执行一个task。第二种是週期性的，例如每一秒钟执行一个task。第三种是指定某个时间执行一个task。当然这些应用会包装在`Timer`或是`ScheduledThreadPoolExecutor`中，但是底层都还是用thread去完成的。
5. Daemon，或是称之为service。有些时候我们需要某个thread专门去等某些event发生的时候才会去做对应的事情。例如写server程式，我们会希望有一个thread专门去听某个port的连线讯息。如果我们写一个queue的consumer，我们也会开个thread去听message收到的event。

我们在写java程式中，其实我们每天都在跟thread共舞，即便我们没有直接产生这些thread，但是我们所用的framework可能都已经建构在multi-thread的环境之上，因此我们更需要瞭解multi-thread的课题。下个章节我们来讨论thread之间的synchronization的问题。

> 一般而言，线程总是作为一个处理流程，可以分为IO绑定线程和CPU绑定线程，一个IO总是绑定一个线程，这主要是因为系统调用的虽然有阻塞式和非阻塞式调用，但如果要做到系统通知数据到来，以前只能使用一个线程阻塞的read某文件描述符或每个一段时间非阻塞的调用read。幸运的是，后面出现select, poll, epoll等多种系统调用，这些系统调用可以同时监听多个文件描述符的状态。这导致线程可以多路IO复用，同时和多个IO绑定。
