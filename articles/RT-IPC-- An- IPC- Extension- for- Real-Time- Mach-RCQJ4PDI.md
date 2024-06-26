---
tags: []
parent: 'RT-IPC:  An  IPC  Extension  for  Real-Time  Mach'
collections:
    - IPC
version: 12754
libraryID: 1
itemKey: RCQJ4PDI

---
# RT-IPC:  An  IPC  Extension  for  Real-Time  Mach

## Abstract

IPC 机制为微内核提供了可扩展和灵活性，但不足以满足实时条件下的时间约束。文展提出 RT-IPC，降低优先级反转，优化实时应用的 CPU 利用率。

## Introduction

Mach 提供了基于端口的 IPC 机制，但它的设计与优化是为了提供所有应用程序公平的服务。IPC 的速度低，导致消息需要排队，但优先级反转机制并没有与 RT-IPC 相结合，因此在 client 与 server 之间的通信中的优先级反转问题越来越严重。

## Motivation

### Overview of Mach IPC

<span style="color: rgb(37, 37, 37)"><span style="background-color: rgb(255, 255, 255)">IPC 通道称为端口。一个端口是由保存消息的队列组成的单向通道。 一条消息是类型化的数据集合。</span></span>

任务具有端口命名空间，只有当任务有用正确的端口权限时，才可以对端口进行控制。只有一个任务具有端口的接收权限。多个任务可以拥有同一个端口的写权限。

### Basic Issues

#### Message Queueing Order

FIFO 导致严重的优先级反转。

#### Priority Hand-off

接收线程或接收消息的优先级不一致会导致优先级反转。

#### Priority Inheritance

发送线程的优先级继承接收线程的优先级。

#### Message Distribution

接收方处理线程数量较少时，需要选择线程，并将线程优先级提高。

## Real-Time IPC Model

RT-IPC 不保证顺序。

### Port and Receiver Association

RT-IPC 支持两种类型的传输模式：1. 同步传输，发送者阻塞，直到收到接收者返回的回复。2. 异步传输，发送者不必阻塞等待接收者的回复。两种模式下，接收者必须在开始传输之前声明自己为接收者。发送者将根据端口的属性来维护自己的优先级。

### Message Buffer

在通信之前预分配消息缓冲区。

### Port Attributes

1.  Message Queueing：维护消息队列的顺序
2.  Priority Hand-off：控制接收者的优先级
3.  Priority Inheritance：server 继承发送线程的优先级
4.  Message Distribution：当多个接收者运行时，选择合适的接收者

### Integration with Synchronization

……

## 总结

通过合理设置发送方、接收方、消息的优先级来避免优先级反转，保证在实时环境下的 IPC 不会被优先级反转影响，导致时间开销增大。
