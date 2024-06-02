---
tags: []
parent: ""
collections: []
version: 11823
libraryID: 1
itemKey: C799K8LP

---
# A Fast Mach Network IPC Implementation

Mach：微内核

## Introduction

在同样的硬件上，Mach 的网络 RPC 比其他系统的 RPC 慢

要实现基于非共享内存的多处理器需要实现快速的网络 IPC。

非共享内存多处理器有高吞吐量和超低延时的互联，建立在慢 IPC 上不合适。

## Issues and Implementation

### Message size and latency

对于小规模消息，软件延迟是主要的开销。

对于大规模消息，网络吞吐和依赖于数据的软件成本（在缓冲区之间复制数据）是主要的。

### Large messages

*   Avoiding copies via shared buffers

    *   单个共享缓冲区无法避免任务之间的干扰
    *   若对共享缓冲区进行分片，则会限制可发送的消息大小，也不能避免而已用户程序
    *   必须要等待缓冲区的数据被设备发送出去后才可以进行修改

*   Avoiding copies via virtual memory mapping

    *   将用户数据所在的地址映射到内核地址空间中，当收到数据时，将地址空间映射到接收方的地址空间中

    *   避免用户修改发送的数据方法：

        *   在内核不返回
        *   修改页面的访问权限

*   Avoiding copies on receiving using virtual memory mapping

    *   数据放到匿名页面中，再映射到用户地址空间中

*   Integration with local IPC implementation

    *   将远程IPC集成到内核中的本地IPC会导致内核复杂

*   Port names, capabilities, and reference counts

    *   通过全局端口描述符来控制 IPC 权限，以及目标方
    *   全局端口描述符中包含了地址信息
