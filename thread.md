# How to use

废话不多说，直接来看java thread怎麽使用

```java
new Thread(() -> {
    System.out.println("hello thread");
}).start();
```

都已经java8的时代了，我直接用java8的语法来介绍java thread。在上面的程式中产生了一个新的thread，thread的constructor是一个实作`java.lang.Runnable`的物件。当呼叫`start()`此method时，则会启动这个thread，并且执行`Runnable#run()`。在java8中有好用的lambda，我们直接用lambda来实作那个runnable。当然你也可以用传统的产生一个新的class来实作`Runnable`。因为这个文件不是要教你怎麽写java，我就先假设你已经了解写java的基本常识，如果你到这边还不懂我在写什麽，那建议可以先拿一本java入门书来了解java最基本的编程知识。

# 使用时机

通常有以下几种情况会用thread

1. IO相关的task，或称IO bound task。如果同时需要读很多个档案，或是同时要处理很多的sokcet connection，用thread的方法去做blocking read/write可以让程式不会因为等待IO而导致什麽事情都不能做。
2. 执行很耗运算的task，或称CPU bound task。当这种task多，我们会想要使用多个CPU cores的能力。单执行绪的程式只能用到single core的好处，也就是程式再怎麽耗CPU，最多就用到一个CPU。当使用multi-thraed的时候，就可以把CPU吃饱吃满不浪费。
3. 非同步执行。其实不管是IO-bound task或是CPU-bound task，我们应该都还是要跟主程式做沟通，所以通常的概念都是开一个thread去做IO bound或是CPU bound task，等到他们做完了，我再把结果拿到我的主程式做后续的处理。当然新开thread不是唯一的方法，我们稍后的章节会陆续提到很多非同步执行的策略跟方法。
4. 排程。常见的排程方法有三种。第一种是delay，例如一秒钟后执行一个task。第二种是週期性的，例如每一秒钟执行一个task。第三种是指定某个时间执行一个task。当然这些应用会包装在`Timer`或是`ScheduledThreadPoolExecutor`中，但是底层都还是用thread去完成的。
5. Daemon，或是称之为service。有些时候我们需要某个thread专门去等某些event发生的时候才会去做对应的事情。例如写server程式，我们会希望有一个thread专门去听某个port的连线讯息。如果我们写一个queue的consumer，我们也会开个thread去听message收到的event。

我们在写java程式中，其实我们每天都在跟thread共舞，即便我们没有直接产生这些thread，但是我们所用的framework可能都已经建构在multi-thread的环境之上，因此我们更需要瞭解multi-thread的课题。下个章节我们来讨论thread之间的synchronization的问题。

