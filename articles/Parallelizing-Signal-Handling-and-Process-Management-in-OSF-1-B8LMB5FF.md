---
tags: []
parent: ""
collections: []
version: 11887
libraryID: 1
itemKey: B8LMB5FF

---
# Parallelizing Signal Handling and Process Management in OSF/1

## Introduction

原始的 4.3BSD-Reno 为单处理器设计，使用中断提供的同步机制在 SMP 环境下失效。

由于内核代码不支持并行，因此与进程管理和信号处理的系统调用需要很长时间执行。

增加合适的锁和其他同步机制，使得这些系统调用可以在任意处理器上运行。

## Background

### Related Unix system calls

与进程管理相关的系统调用：fork、exec、wait、exit 等。

#### Signals

软件中断：

*   提示超出资源限制
*   挂起或取消挂起进程
*   提示没有访问终端权限的进程想要访问输入输出

#### Process groups

#### Job control

#### Sessions

### Synchronization overview

需要两类锁保证不同处理器之间的同步

*   simple lock（spin lock）
*   blocking lock（rw lock/mutex lock）

使用这些同步原语的目标：

*   最小化源代码级改动
*   自适应不同的硬件配置（单处理器、多处理器）
*   使用引用计数最小化动态锁范围
*   最大化应用程序可见接口的并行，以及内核
