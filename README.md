# Introduction
Thread(线程，或称执行绪)是一个美妙的东西，它允许你在同一个address space(定址空间)撰写concurrency(并发)以及parallelism(并行)的程序。通常跟它会拿来比较的是process(进程)，跟thread不一样的是，process之间会是在不同的定址空间，因此可以做到比较高度的隔离，但是缺点就是比较难在process之间沟通。因此，thread我们会称为较lightweight的process。

在Java当中一个JVM就是一个process，在JVM中可以产生非常多的threads。Java thread早在java的最早版本就存在，在那个multi-threading还没有很成熟的年代，java thread的简单好用让大家开始发现写multi-thread不是什么难事。

这本书主要目的在于：
1. 使读者了解什么是线程，如何简单的使用线程
2. 线程间如何通信
3. 如何使用线程池
4. 如何写异步程序

本书内容是以Java8为基础，除了在多处使用lambda之外外，也介绍了java8才有的CompletableFuture。如果还未写过Java8的读者，相信基本观念还是会有帮助。

