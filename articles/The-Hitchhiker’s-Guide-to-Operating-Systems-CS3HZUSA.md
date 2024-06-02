---
tags: []
parent: 'The Hitchhiker''s Guide to Operating Systems'
collections:
    - osdi
version: 11830
libraryID: 1
itemKey: CS3HZUSA

---
# The Hitchhiker’s Guide to Operating Systems

**核心观点**：从状态转换的视角来展开操作系统教学。将计算机系统的所有组成部分- -包括硬件、应用程序以及连接它们的操作系统- -视为状态转换系统。

## State Machines：First-class Citizens of Operating Systems

将用户级应用与内核都视为状态机，OS 为状态机管理者。

**Program as a state machine**：在没有系统调用的情况下，用户程序（状态机）是一个“封闭世界”，只能对内存和寄存器的值进行算数和逻辑运算。通过系统调用访问外界。

**Bare-metal as a state machine**：无限循环的状态机，通过端口或 MMIO 访问外界，并且可以被打断。

**Computer system stack on state machines**：每个程序都是状态机，其初始内存布局和状态转移由二进制可执行程序指定。

**状态机与虚拟化**：将状态快照保存起来，并能够根据状态快照恢复执行，使得从用户程序的视角来看，程序在连续执行。

从状态机的视角看，系统提供的接口为操作状态机的功能集合。
