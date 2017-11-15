# Synchronization

在执行了thread之后，下个要问的议题就是要怎麽在thread之间去做synchronization(同步)。我们分以下几个议题来讨论:

1. [Resource Sharing](resource_sharing.md): 如果多个threads同时存取变数该如何解决? 是否可以同时只有我这个thread允许修改某一个resource?
2. [Flow Control](flow_control.md): 一个thread可否等到另外一个thread完成或是执行到某个状况的时候才继续往下做?
3. [Message Passing](message_passing.md): 我可否把我一个thread的output当作另外一个thread的input?


