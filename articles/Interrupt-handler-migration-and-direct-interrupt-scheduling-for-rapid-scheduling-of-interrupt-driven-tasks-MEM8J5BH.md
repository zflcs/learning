---
tags: []
parent: 'Interrupt handler migration and direct interrupt scheduling for rapid scheduling of interrupt-driven tasks'
collections:
    - 调度
version: 16499
libraryID: 1
itemKey: MEM8J5BH

---
# <span style="color: rgb(51, 51, 51)"><span style="background-color: rgb(255, 255, 255)">Interrupt handler migration and direct interrupt scheduling for rapid scheduling of interrupt-driven tasks</span></span>

## Abstract

两种旨在最小化高优先级中断驱动任务的调度延迟的技术

1.  中断处理程序迁移（IHM）：将中断处理程序迁移到相应的目标进程，避免额外的上下文切换，提高中断处理程序生成的数据的缓存命中率
2.  直接中断调度（DIS）：在中断到达和目标进程之间建立紧急中断处理路径（将中断处理划分为不可抢占和可抢占部分）

调度延迟减小 82%，在主频为 200MHz 的 arm cpu 上，调度延时低至 30 微秒，中断处理和调度的开销减少。

## Introduction

OS延迟：为从中断到达到重新激活等待中断的进程之间经过的时间

在 CPU 不够快和不够强大时，减少 OS 延迟对于减少开销很重要。

中断处理程序不仅可以在中断上下文中执行，也可以在线程上下文中执行，中断处理程序不再被视为不可抢占的部分。（中断时只做最少的处理）

使用的两种技术对中断延迟的示意图：

![\<img alt="" data-attachment-key="T53P3HKC" width="924" height="437" src="attachments/T53P3HKC.png" ztype="zimage">](attachments/T53P3HKC.png)

**如果提前对每个中断和要被它重新激活的目标进程之间的相互关系进行预处理和维护，一旦中断到来，就可以立即识别它的紧急性，如果是紧急的，也可以立即识别下一个要运行的进程。**

## Related Work

1.  让 linux 完全可抢占
2.  减少不可抢占段的数量和大小

**spinlock-based 方法**：在持有 spinlock 时，不允许抢占，其他部分可以抢占，但会导致持有 spinlock 时，高优先级的任务不能抢占。

**spinlock-to-mutex 方法**：同样的问题，高优先级任务尝试获取 Mutex 时，会导致至少两个上下文切换开销

**delay locking 方法**：预测中断和持有锁的时间，对可抢占和不可抢占的部分进行重排序

**Lock-Breaking Linux**：将较长的非抢占部分拆分为较小的部分

### Interrupt Handler Thread

中断处理程序和 softIRQ 处理程序的中断上下文执行的优先级高于任何其他进程，因此中断上下文可以抢占任何线程上下文。在中断上下文期间，内核不允许抢占。（中断处理程序和 softIRQ 处理程序被视为不可抢占部分，最高优先级进程）

改进：

以前在 Interrupt 上下文中执行的大多数操作都可以移动到 Thread 上下文中。（但会导致额外的上下文切换）

IHM 将中断处理线程的上下文迁移到目标进程的上下文中

LRP（Lazy receiver processing）：原本在中断上下文中处理的协议处理都是在目标进程的上下文中懒散地执行。只针对网络中断。

### Privious Works on Interrupt handling

1.  interrupt-triggered software prefetching

## Interrupt Handling Migration（IHM）

### Basic Concept of IHM

![\<img alt="" data-attachment-key="QXMJ7UKF" width="1292" height="641" src="attachments/QXMJ7UKF.png" ztype="zimage">](attachments/QXMJ7UKF.png)

a 中的 prev 是已经处于中断处理程序中，内核不可被抢占，只能等这一次中断处理结束之后，再响应这一次产生的中断（从箭头开始的部分已经是在处理中断了，图中没有画出第一次被打断的示意，直接把中断处理算在了 prev 中）（图中的上下文切换是从调度器的视角来看，因为没有切换线程，所以没有上下文切换，但是打断原本的执行流跳转至中断处理，以及恢复的开销是不能避免的。）

这个图只画了从调度器的角度看到的调度开销，只强调了线程上下文切换的开销，没有体现中断上下文的开销，并且把必要的中断处理划分到了目标上下文中，但是这种方式把文章要达到的效果表现得更加明显。

做法：

1.  处于睡眠状态的目标进程下一次执行的入口点是 schedule 函数，因此可以将中断处理函数迁移至这个固定的入口点。
2.  将中断处理的操作从唤醒中断处理线程改为唤醒目标进程

好处：

1.  改善了缓存命中率，因为中断处理以函数调用的形式直接在目标上下文中进行，没有地址空间的切换，生成的数据的缓存友好
2.  解决优先级反转：
3.  改善中断相关的统计：中断处理占用的时间被划分到了目标进程，更加合理

带来的问题：

1.  如果中断没有唤醒目标进程将会发生什么？
2.  如果多个目标进程等待同一个中断源会发生什么？唤醒最高优先级的进程
3.  如果在前一个进程和目标进程之间插入多个中断处理线程会发生什么？多级 IHM

IHM 机制流程：

![\<img alt="" data-attachment-key="7WEG9EGK" width="852" height="646" src="attachments/7WEG9EGK.png" ztype="zimage">](attachments/7WEG9EGK.png)

灰色表示使能了 IHM 机制，白色部分为原来的流程

![\<img alt="" data-attachment-key="NT9IBF2Z" width="861" height="427" src="attachments/NT9IBF2Z.png" ztype="zimage">](attachments/NT9IBF2Z.png)

即使预测的目标进程错误，相较于原本的中断处理线程技术，没有增加延迟

### Operation of IHM under Multiple Waiting Processes

从多个等待进程中选择一个作为目标进程，选择原则：优先级最高

### Operation of IHM under Multiple Interposed Interrupt Handler Threads

对于存在多个中断处理阶段，这种方法可以减少一次以上的调度开销。

![\<img alt="" data-attachment-key="ELEGV2UB" width="856" height="355" src="attachments/ELEGV2UB.png" ztype="zimage">](attachments/ELEGV2UB.png)  

![\<img alt="" data-attachment-key="JPE5742M" width="861" height="335" src="attachments/JPE5742M.png" ztype="zimage">](attachments/JPE5742M.png)

### Resolving the Priority Inversion Problem Caused by IHM

![\<img alt="" data-attachment-key="PAFU8CWM" width="862" height="375" src="attachments/PAFU8CWM.png" ztype="zimage">](attachments/PAFU8CWM.png)

随着中断处理线程的引入（中断处理线程可以增加优先级属性），除了增加一次调度开销，还可能造成优先级反转问题

## DIRECT INTERRUPT SCHEDULING (DIS)

### Basic Concept of DIS

![\<img alt="" data-attachment-key="P6X89LKN" width="995" height="451" src="attachments/P6X89LKN.png" ztype="zimage">](attachments/P6X89LKN.png)

在中断到达和目标进程之间铺设为紧急中断进程对预留的最短路径，使重要的中断进程对处理不被次要的操作延迟。

![\<img alt="" data-attachment-key="D8Q8QST2" width="1252" height="441" src="attachments/D8Q8QST2.png" ztype="zimage">](attachments/D8Q8QST2.png)

![\<img alt="" data-attachment-key="B9XE94UH" width="1101" height="485" src="attachments/B9XE94UH.png" ztype="zimage">](attachments/B9XE94UH.png)

time-indep 和 time-dep 指以下与时间相关的统计信息，运行时间、修改任务状态等。

## EXPERIMENTS

使用时钟中断以及网络中断进行测试。

使用额外的硬件定时器来生成紧急的时钟中断

三个进程：

1.  Timer task：等待 5 ms 一次的紧急时钟中断
2.  Net task：等待网络包时休眠
3.  Calc task：重复一系列算数运算

### One Process Waiting for a Timer interrupt

同时运行 timer task（优先级高） 和 calc task

目标：在calc任务存在的情况下，最小化相对于定时器任务的OS延迟。

![\<img alt="" data-attachment-key="5INYIQS4" width="858" height="796" src="attachments/5INYIQS4.png" ztype="zimage">](attachments/5INYIQS4.png)

### One Process Waiting for a Network Interrupt

同时运行 Net task 和 calc task，Net task 运行在嵌入式板上，向远端的 server 发送请求，RTT 为 500 微秒，使得任务在发送之后可以进入睡眠也可以保证等待响应的时间不会太长，网络包除开头部信息外，数据长度为 10 字节

![\<img alt="" data-attachment-key="83PV8WTT" width="850" height="865" src="attachments/83PV8WTT.png" ztype="zimage">](attachments/83PV8WTT.png)

### Two Processes Waiting for a Network Interrupt and a Timer Interrupt, Respectively

![\<img alt="" data-attachment-key="CN4RYJPJ" width="848" height="386" src="attachments/CN4RYJPJ.png" ztype="zimage">](attachments/CN4RYJPJ.png)

右图两条垂线中间表示 timer task 正在运行

### 评价

这里减少了内核不可抢占的部分，但是上下文切换和前半段（唤醒中断处理线程或目标进程）还是不可抢占的，而我的做法则只有上下文切换的代码是不可抢占的。
