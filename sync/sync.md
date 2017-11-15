# 线程间同步

在执行了thread之后，下个问题就是要怎么在thread之间去做synchronization(同步)。 我们分以下几个问题来讨论:

1. [共享资源](resource_sharing.md): 如果多个threads同时存取某一个变量该如何解决? 如何保证数据的一致性？
2. [流程控制](flow_control.md): 一个thread可否等到另外一个thread完成或是执行到某个状况的时候才继续往下做?
3. [消息传递](message_passing.md): 我可否把我一个thread的输出当作另外一个thread的输入?
