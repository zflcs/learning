---
tags: []
parent: 'Preventing interrupt overload'
collections:
    - 硕士毕业阅读文献
version: 7370
libraryID: 1
itemKey: IHWL27BS

---
# Preventing interrupt overload

#### 赵方亮笔记

## Introduction

许多中断驱动的嵌入式系统容易受到中断过载的影响。外部中断频繁产生，导致处理器上的其他活动都处于饥饿状态。

## 评论

用硬件降低发送给 CPU 的中断信号频率，防止频繁的中断让 CPU 无法执行其他的任务
