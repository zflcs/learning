---
tags: []
parent: 'ImplementingLightweight Threads'
collections:
    - 任务
version: 11939
libraryID: 1
itemKey: TMZXYIIZ

---
# Implementing Lightweight Threads

## Abstract

*   描述了轻量级线程库的实现
*   在内核支持的线程池上实现进行多路复用
*   由库管理线程池，自动增长/收缩

## Threads Model

在进程内的多个线程编号，在进程外部不可见。

进程内的线程之间可以通过 mutex、条件变量、信号量、rw 锁实现同步；

在不同进程之间的线程（即使相互不可见）可以通过共享内存或者 mapped file 进行同步。

## Threads Library Architecture

LWP：light-weight process
