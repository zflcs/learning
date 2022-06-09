# 高级操作系统

### 读论文

1. 创新点
2. 解决问题
3. 相关的重要工作
4. 有意思的部分
5. 对实验的客观评价
6. 能否改进这个研究，自我思考
7. 自己测试结果

### 对 OS 的重新思考

1. Single-Machine OS/VMM
   1. Memory Management
      1. 多机内存管理
      2. 简化内存管理（swap、paging）
   2. Process Scheduling
      1. 减少 context switch
   3. Synchronization
      1. 新的硬件环境设计新的结合硬件的同步互斥机制
      2. 结合硬件/软件机制避免死锁
   4. File System
      1. 存储介质
      2. 可靠性
      3. 大数据
   5. I/O Device
      1. I/O 设备速度加快，新的驱动、协议栈
2. OS Performance Multi-Core
   1. 加速比：很难达到线性加速比
   2. 吞吐量：内存文件系统 tmpfs ，多个 CPU 访问文件系统，内核锁竞争导致吞吐量下降，需要修改 kernel 的同步机制
3. OS Reliability
   1. Threat Analysis：purpose、vulnerabilities、who、prevent、cost
   2. 当前威胁：语义漏洞、linux 内核漏洞，但是发现漏洞的难度增大（也许是因为 C 语言本身的原因）
   3. security
   4. fault tolerance
4. OS Correctness
   1. 对问题的定义，建立规范，程序语义与规范是否一致
   2. 代码加载，控制逻辑，跳转、异常处理、进程线程切换怎么表述
   3. 对数据的改变是否和规范一致，指针运算怎么处理
   4. Device 的异步性对程序的分析造成的影响

### OS 架构

1. Simple Kernel		MS-DOS
2. Monolithic Kernel   单体内核，层次化设计
3. Micro Kernel    微内核、消息传递、效率低
4. Exokernel  外核、虚拟硬件，然后针对某个应用定制 OS
5. History
   1. MULTICS
   2. THE 分层架构设计
   3. UNIX   Kernel 通过 file 对其他的进行抽象
   4. Mach & L4
   5. Xok + ExOS：解决性能问题

### Virtual Machine Monitor

