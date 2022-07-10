### Green Threads

#### Example

- 主要是关于建立好自己的栈，然后不使用函数调用的方式，而是直接在栈中存放数据，之后再模拟函数调用执行来展示了简单的上下文切换过程

#### Stack

- 固定大小的栈，会导致栈溢出

- 可增加的栈，当快要溢出的时候，则会在堆中找到一块更大的空闲区，然后把原先的栈里面的内容复制到新的栈中，但是栈里面经常会存放一些指针，显然这是一个自引用结构，会导致移动之后，所有的指针都必须要更新

#### Stack Frame

- 主要介绍了 stack 的布局

#### Implementation

定义了 RUNTIME、Thread 以及 ThreadContext 数据结构

- RUNTIME：主要是模拟内核，提供run、return、yield 等接口
    - 从当前的 GreenThread 返回时，直接令当前的 GreenThread 的状态为 Available，表示这个 GreenThread 已经为空，可以分配，之后调用自身的 yield 接口，来调度下一个 GreenThread
    - yield 接口：从就绪队列中找到一个 GreenThread，然后切换上下文去，开始执行下一个 GreenThread
    - run 接口：循环调用 yield 接口，一开始时会找到一个就绪的 GreenThread 去执行，执行完之后会返回（这里的返回是指的封装了 RUNTIME.t_return 的 Guard，而不是简单的 return 关键字），返回之后会回到 RUNTIME.t_yield，之后会进行下一次调度，实际上是不会回到第 0 个 GreenThread（或者称之为 idle_green_thread），等到所有的 GreenThread 执行完毕之后，就意味着整个过程已经结束了，就直接调用 std 的 exit 函数来退出这个 run 函数
- Thread：封装了 id、stack、ThreadContext 上下文以及 state 状态
- ThreadContext：需要程序员根据机器的架构来自定义，但是这个上下文结构是 GreenThread 自己能够感知到的，因此在切换时不需要保存 CPU 的所有信息，如果让内核来负责上下文切换，内核是无法感知到 GreenThread 内部的，因此只能是一刀切，全部保存

#### Drawbacks：

- 栈的大小不一定是固定的，但是可变大小又会带来很多问题
- 上下文需要根据不同的平台来实现，只能是定制的，难以跨平台，很难做到零成本抽象

