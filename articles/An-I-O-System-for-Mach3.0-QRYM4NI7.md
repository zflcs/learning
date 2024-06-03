---
tags: []
parent: ""
collections: []
version: 11914
libraryID: 1
itemKey: QRYM4NI7

---
# An I/O System for Mach3.0

## Introduction

1.  将驱动代码分离成与设备芯片无关的代码以及与设备强相关的代码
2.  在用户态映射设备，允许应用程序直接操控设备
3.  Mach内核提供了位置透明的设备管理，可以通过Mach的处理器间通信(IPC)设施进行访问。

由于位置透明，因此可以不仅仅只控制本地的设备，而且可以控制远端的设备，因此设计了基于 IPC 的设备接口。

## User-Level Device Management

内核将设备的寄存器映射到用户空间中，将设备中断映射到用户线程。

Devices can be managed from user-level by vectoring all device interrupts out to an application's thread.

通过将所有设备中断矢量化到应用程序线程，可以从用户级别管理设备。

<span style="background-color: #ffd40080">不太明白这句话的实际做法，应该是说产生了中断后切换了特权级，但是直接执行了在用户地址空间的代码，这里应该是只有单页表</span>
