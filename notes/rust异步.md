### rust 异步机制

async：零成本，无需堆内存分配和动态调度，显著降低内存和 cpu 开销，支持比系统线程多几个数量级的任务

不允许在 trait 中声明 async 函数

async 将一段代码转化为一个实现了 Future trait 的状态机，阻塞的 Future 会放弃对线程的控制，从而允许其他的 Future 来运行



##### 异步函数语法

- async fn do_something(){}

- async fn 返回 Future，Future 需要一个执行者来运行



futures::executor::block_on 阻塞当前线程，知道提供的 Future 运行完成

.await 不会阻塞当前线程，而是异步的等待 Future 的完成



##### 执行 Future 和任务

Future 代表一种可以检验其是否完成的操作

Future 可以通过 poll 函数取得进展，对于 Future 只能使用 poll 来推动它，直到它产生一个值

当 Future 准备好取得更多进展时调用 waker 的 wake() 函数



##### wake() 函数

当 wake() 函数被调用时，执行器将驱动 Future 再次调用 poll 函数，通过 wake() 函数，执行器就确切的知道哪些 Future 已经准备好 poll() 的调用

没有 wake() 函数，执行器就不知道特定的 Future 何时能取得进展（只能不断的 poll）



##### Future 执行者（Executor）

.await 把问题推倒了上一层，顶层 async 函数返回的 Future

Future 执行者会获取一系列顶层的 Future，通过在 Future 可以有进展的时候调用 poll，来将这些 Future 运行至完成

执行者将 poll 一个 Future 一次，以开始

当 Future 通过调用 wake() 表示它们已经准备好取得进展时，它们会被放到一个队列中，然后poll再次被调用，重复这个过程，知道 Future 完成



##### async 和 .await

rust 的特殊语法，在发生阻塞时，它让放弃当前线程的控制权成为可能，这就允许在等待操作完成的时候，允许其他代码取得进展

async 的两种用法：async 函数、async 代码块

执行途中能够暂停执行（线程在执行其他的工作），然后恢复执行的能力是 async 所独有的，await 依赖于这种特性，只能在 async 里 

async 生命周期：async 函数的参数是引用或者其他非 'static 的，Future 就会绑定到参数的生命周期上

存储或传递 future：（async 的函数在调用之后立即 .await 不会出现问题）

- 把使用引用作为参数的 async fn 转为一个 'static future，在 async 中将参数和 async fn 的调用捆绑到一起

async 块和闭包都支持 move

- async move 块会获得其引用变量的所有权，允许其比当前所在的作用域活得长，但是放弃了与其他代码共享这些变量的能力



Pin 与 Unpin 标记一起工作

Pin 保证实现了 !Unpin 的对象永远不会被移动

 

join! 多个 Future 同时执行

select! 选择一个 Future