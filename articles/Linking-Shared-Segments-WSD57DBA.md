---
tags: []
parent: ""
collections: []
version: 12026
libraryID: 1
itemKey: WSD57DBA

---
# Linking Shared Segments

## Abstract

比起通过消息或文件的通信方式，共享内存更简单、快速且浪费的空间更小。但在 Unix 中共享的机制较少。

Hemlock 与 Unix 向后兼容，使用了 mmap 系统调用以及内核维护的虚拟地址和文件之间的映射关系，并且引入了范围链接的概念，避免与已有的共享存在的命名冲突。

## Introduction
