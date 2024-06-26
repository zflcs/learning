---
tags: []
parent: 'Anonymous RPC: Low-Latency Protection in a 64-Bit Address Space'
collections:
    - IPC
version: 12692
libraryID: 1
itemKey: 39KFKVX3

---
# Anonymous RPC: Low-Latency Protection in a 64-Bit Address Space

## Abstract

传统系统基于地址空间隔离提供内存保护，但及时经过优化，在进行跨域 RPC 时，地址空间切换的开销仍然很大。

文中使用了 `_anonymity_ `，而不是使用硬件页表来提供保护。逻辑独立的内存段被随机放置在同一个地址空间和保护域中。使用 64 位的虚拟地址，<span style="color: rgb(37, 37, 37)">进程不太可能通过意外或恶意内存探测来定位任何其他段；</span>

## Introduction

使用虚拟内存硬件虽然提供了灵活性和保护机制，但开销很大（增加了上下文状态的数量以及上下文切换的开销）。对 IPC 的开销设定了严格的下限。

IPC 需要两次上下文切换，限制了两个逻辑上独立，但紧密合作的模块划分成单独的保护域的可能性。

上下文切换中的软件开销随着处理器的发展而减小，但硬件开销没有。因此对极细粒度的 IPC 构成了重大的障碍。

一种解决思路：使用轻量级的线程，共享用一个保护域，但线程不安全，即使在可信环境下，也是脆弱的。

目标：结合 UNIX 进程的保护机制以及线程的速度，使 IPC 开销减小，能够大量使用。<span style="color: rgb(37, 37, 37)">这可以允许比现在可行的更细粒度的应用程序交互，并且大型软件系统可以被构造为协作进程组而不是单个整体实体。</span>

方法：<span style="color: rgb(0, 0, 0)">_anonymity_，</span>限制段的映射关系

在 32 位地址空间下不可行，可以进行穷举，64 位地址空间足够大，因此可行。可以使用随机映射的未保护的页来做共享内存。anonymity 可以被用于一种通用的保护机制，给共享相同地址空间中的进程提供安全的保护机制。共享相同地址空间，上下文切换只需要保存寄存器和栈，与轻量级线程相同。

## Implementing Anonymous Protection

1.  需要随机地址分配（若某个段是匿名的，则没有权限的进程是无法发现或导出它的。），使用了加密系统。

## Anonymous RPC（ARPC）

### Maintaining Anonymity During Communication

正常的 <span style="color: rgb(0, 0, 0)">procedure call</span>，调用者需要知道 <span style="color: rgb(0, 0, 0)">procedure 在被调用者代码段的地址，并且需要告诉被调用者自己的地址来返回，这与匿名是冲突的。为了保持匿名，需要一个“中介”，管理和控制数据流，无需泄露任何一方的地址。</span>

通过只执行的代码段进行跳转，保证匿名性，配合只执行跳转表，将入口作为立即数，带来的开销是额外的跳转。（被调用者需要提供只执行代码的跳转表）

通常的调用数据共享在同一个栈上，为了保证匿名性，需要单独的协议来进行数据传输，切换栈，清除可能破坏匿名性的数据寄存器。

### ARPC Protocols

针对不同的可信方，使用不同类型的 RPC。

#### Binding

server 发起，先注册入口点，把 procedure 的名称和地址告诉 RPC manager。client 可以通过 connect 来告诉 RPC manager 想要访问的 procedure 的名称，如果通过权限检查，则 RPC manager 生成绑定，生成不同的中介（只执行跳转表、内核陷入口、server 的直接入口），并准备好执行栈，针对每个 procedure，静态分配一个执行栈。

#### ARPC，Mutual-Distrust

    The protocol is outlined in Figure 1:

                    push arguments and return address on client stack
                    enter intermediary
                    save return address
                    save registers
                    clear registers
                    save address of client stack
                    copy arguments to server stack
                    switch to server stack
                    leave intermediary to server procedure

                        execute server procedure
                        
                    enter intermediary
                    copy return data to client stack
                    switch to client stack
                    restore saved registers
                    clear unsaved registers
                    leave intermediary to client return

    Figure 1: ARPC Protocol, Mutual Distrust Between Client and Server

对于较多的数据，由于 client 的所有地址，server 都能够访问，因此传输参数时，将数据保存在与私有数据分开映射的段中，并且传递指针。

#### ARPC，Server Not Malicious

    The protocol is in Figure 2.

                    save live registers
                    push arguments and return address on stack
                    call through intermediary to server

                    push address of client stack
                    switch to server stack
                    execute body of server procedure
                    pop address of client stack
                    push return data onto client stack
                    clear sensitive registers
                    return through intermediary to client procedure

                    restore live registers

    Figure 2: ARPC Protocol, Server Not Malicious

相较于互相不信任的方式，这种方式会减少寄存器上的信息的复制

#### ARPC，Neither Side Malicious

    both client and server are trusted to be non-malicious, in Figure 3.

                    save live registers
                    push arguments and return address on stack
                    call directly to server procedure

                    push address of client stack
                    switch to server stack
                    execute body of server procedure
                    pop address of client stack
                    push return data onto client stack
                    return directly to client procedure

                    restore live registers

    Figure 3: ARPC Protocol, Neither Side Malicious

不需要经过中介，相较于函数调用，这里需要切换栈。

### Service Management

在 client 线程上运行 server 的 procedure。当 server 崩溃时，为了避免 client 线程死去，RPC 需要返回错误代码。有中介来保证客户端线程的恢复。如果多个线程需要使用相同的 RPC，则必须在外部进行同步。

## Uses of Anonymous RPC

### User-level ARPC

主要用于避免意外破坏数据，可以将不同的线程使用不同的堆，放到不同的保护域中。

### ARPC as an Operating System Base

能够加速用户进程之间的通信，帮助对内核进行分解。

### Limited Uses of ARPC

与传统的硬件保护机制相互并存，但适用于对速度要求较高的环境下。

## Some Problems and Disadvantages of ARPC

1.  初始化较慢，每个段都必须随机映射，执行时进行重定位。
2.  地址空间稀疏，难以管理
3.  使用进程标识符标记的 TLB 和虚拟缓存将降低上下文切换的开销，这种情况下基于传统的保护机制将比 ARPC 更有竞争力

# 总结

基于 64 位虚拟地址，并且使用较强的随机性的地址空间分配算法，探测到匿名域所需要的开销很大，因此，可以将匿名域与其他的域放到同一个地址空间中，来避免上下文切换。
