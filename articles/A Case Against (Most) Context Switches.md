### A Case Against (Most) Context Switches



**在一个物理核上增加大量的硬件线程，使用软件进行管理。**

上下文切换过于频繁，切换开销巨大。即使新的 CPU 上每个核有很多硬件线程，但 OS 仍然将这些硬件线程视为独立的个体，分配软件线程执行。

#### 优点：

##### 没有中断

硬件线程使用 mwait 阻塞在某个内存地址上，等待的事件源只需要写对应的等待的地址

##### 快速 IO

内核不再需要轮询的线程，直接让硬件线程 mwait IO 事件队列

##### 优化系统调用和 VM-exits

使用专门的硬件线程来处理

##### 在内核中访问所有寄存器

原来内核中不访问浮点寄存器和向量寄存器，因此上下文切换时，这些寄存器不需要保存。内核和应用程序运行在不同的硬件线程上，因此可以使用所有的寄存器

##### 快速的微内核和代理容器

硬件线程直接运行相关的服务，应用程序直接启动对应的硬件线程，进行通信，不需要内核的调度器

##### 更简单的分布式编程



#### 软硬件接口

##### 硬件线程

- 三种状态：runnable, waiting, or disabled；硬件系统通过执行 monitor/mwait 指令进入 waiting 状态；处于 disabled 状态的硬件线程只能等待其他硬件线程唤醒
- 一个硬件线程可以同时监听多个内存地址
- monitor 可以监视任意地址，包括 DMA，可在任意权限使用
- start、stop 将硬件线程映射到某个虚拟线程
- rpull 和 rpush 用于读取处于 disabled 状态的硬件线程寄存器

##### TDT

- 保存 vtid 与 ptid 之间的映射关系，记录相关的硬件线程之间的控制权限，除了以权限环的形式，还可以在专门的硬件线程上运行硬件线程管理程序

##### 线程状态

- 使用大规模的 regfile 存储硬件线程状态；或者使用缓存（L2-cache 或者 L3-cache）



##### 思考：

1. 上述的硬件线程通过 monitor 指令，实际上还是通过共享内存进行通信。缺点是硬件线程不能使用，但是响应将会较快。这里可以在软件上，用协程实现，一个协程轮询的读取某个共享的变量，可以让权，其他协程对这个变量进行读写从而唤醒阻塞的协程，缺点是协程切换的开销。如果实现为硬件的协程执行环境，也可以实现文章的思路，两者的区别应该只在于执行环境包括的寄存器数量大小。
2. 如果要实现硬件的协程执行环境，硬件协程的寄存器状态应该如何保存？硬件协程与软件定义的协程之间的映射关系？软件定义的协程在硬件协程上的调度顺序？
3. 硬件实现后，软件上介于同步和异步之间的临界的函数将会消失。
4. 未完待续。。。
