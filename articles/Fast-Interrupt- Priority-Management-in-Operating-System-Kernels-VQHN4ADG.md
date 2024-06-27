---
tags:
    - 在中断实际发生时才修改硬件中断屏蔽位
parent: 'Fast Interrupt  Priority Management in Operating System Kernels'
collections:
    - 中断
version: 12814
libraryID: 1
itemKey: VQHN4ADG

---
# Fast Interrupt  Priority Management in Operating System Kernels

## Abstract

提出了一种在 OS 内核中控制处理器中断状态的技术。

在 IPC、调度和内存管理等模块内，单处理器和多处理器在关键部分可以通过禁用中断来避免死锁和数据冲突。但修改中断屏蔽位所需要的周期逐渐增加。

## Introduction

中断处理功能没有跟上处理器发展的速度。OS 利用中断来控制调度和 I/O，使用中断屏蔽来保证跨中断的资源共享。

文中提出的乐观中断保护机制减小了中断屏蔽的开销但保留了中断模式的语义，减小了 50% 的中断管理开销，使 IPC 加速了 5.3%。

## Interrupt management

OS 依赖中断来响应异步事件，中断屏蔽是一种针对异步事件的数据保护机制。

    ______________________________________________________________________________
    |_Machine_______________|Processor_________________|__Instructions__|_Cycles_|
    | Luna88k               | Motorola 88100 (25MHz)   |       68       |    96  |
    | Flamingo              | Alpha 21064 (150MHz)     |       25       |    21  |
    | DECstation 5000/120   | R3000 (20MHz)            |       65       |    80  |
    | DECstation 5000/200   | R3000 (25MHz)            |       14       |    14  |
    | 386 PC                | Intel 386DX-25 (25MHz)   |       51       |   179  |
    |_Gateway_66V___________|_Intel_486DX2-66_(66MHz)__|_______51_______|___279__|

    Table 1:  Overhead of  changing the interrupt mask.  Cycle  counts
    are estimated, assuming no cache  misses.

在单处理器上，中断屏蔽可以很好的工作，不需要其他的同步机制。中断屏蔽模式的一个优点是延迟敏感的事件可以抢占长期运行的低优先级的活动。在传统的机器上，中断屏蔽只需要几个周期，但修改硬件中断级别的时间却没有随着处理器速度加快而减小。

## Optimistic interrupt protection

乐观中断保护机制利用了在通常情况下，中断不会发生在临界区中的事实。当处理器执行内核，进入临界区时，设置软件中断屏蔽位标识需要屏蔽中断，不改变硬件中断屏蔽位。在特殊情况下，低优先级的中断发生了之后，中断处理 prologue 构造一个中断 continuation，根据软件中断屏蔽位更新硬件中断屏蔽位，并马上返回被打断的活动。这种方式只有在实际发生了中断时才会修改硬件中断屏蔽位，避免了多余的修改，并且能够把修改中断屏蔽位的操作从 fast path 中移出。

中断 continuation 是一个包含了中断延迟时的系统状态的数据结构，包含了中断延迟处理的足够信息。在临界区尾部，处理器检查中断 continuation，若没有，则直接返回正常的执行流，若存在，则处理完后，恢复硬件中断屏蔽，返回正常执行流。

与传统中断控制相比，乐观中断保护将屏蔽的中断处理推迟到临界区尾部，要求中断处理 prologue 执行安全的代码。开销仅仅是在进入临界区和退出临界区时，增加检查两个变量的逻辑。

## Performance

在多处理器上，优化的效果不如单处理器。

    _____________________________________________________________________________
    |_Machine_____________|Conventional_|__Optimistic_|_Speedup_|_Cycles__saved_|
    | Luna88k             |    4400     |    4225     |  5.3%   |      175      |
    | DECstation 5000/120 |    2140     |    1840     |   14%   |      300      |
    |_DECstation_5000/200_|____1234_____|____1198_____|__2.9%___|______36_______|

    Table 3:  IPC performance.   Shown are the cycles for  a null RPC
    with optimistic and conventional interrupt  management.  One cycle
    on the Luna88k is 40  nanoseconds, so IPC latency is reduced  by 7
    microseconds from 176 to 169.   The cycle times on the  5000/120
    and 5000/200 are 50 and  40 nanoseconds respectively.

## Related Work

OS 的基本设计决策之一是如何处理同步和异步事件之间的协调：（interrupt mask、non-preemptable、lock-free synchronization）

在 non-preemptable 方法中，同步时间和异步事件的处理都不能被打断，这要求所有的处理都必须很短，不能阻塞。

lock-free synchronization 对硬件有额外的要求。
