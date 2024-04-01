---
tags: []
parent: 'Refactor, Reduce, Recycle: Restructuring the I/O Stack for the Future of Storage'
collections:
    - 硕士毕业阅读文献
version: 9686
libraryID: 1
itemKey: GCUBNSI5

---
# Refactor, Reduce, Recycle: Restructuring the I/O Stack for the Future of Storage

问题：

**随着 NVM 和 PCM 等新设备的出现，硬件性能不再是瓶颈，因此必须重新思考与存储相关的软件的性能。**

三个基本原则：

1.  refactor：将设置存储策略与执行存储策略分开
2.  reduce：设计人员应该对硬件和软件进行协同设计，以消除软件的干扰，以便系统能够充分利用 NVM 的原始性能
3.  recycle：利用现有技术和软件组件，减少设计工作量，从而简化采用过程

重构了 I/O 堆栈，从常见读写操作的代码路径中删除可信软件（操作系统和文件系统）。

Moneta 驱动：

1.  User-space driver：LibMoneta 提供了对 Moneta 硬件的直接访问，不需要调用操作系统
2.  Virtualized hardware interface：每个进程都使用私有逻辑接口来访问 Moneta，允许多个应用程序同时使用 Moneta
3.  Hardware permission checks
4.  Permissions manager in the kernel
