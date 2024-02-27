---
tags: []
parent: 'Chisel-based Implementation of High-performance Pipelined Priority Queue Generator'
collections:
    - 硕士毕业阅读文献
version: 8225
libraryID: 1
itemKey: 3WJDZ5EK

---
# Chisel-based Implementation of High-performance Pipelined Priority Queue Generator

恒定的两个时钟周期完成入队和出队操作；

## Introduction

介绍了优先级队列的重要性

几种常见的优先级队列：

1.  binary-tree-of-comparators-based priority queue
2.  systolic-array-based queue
3.  shift-register-based priority queues
4.  calendar queues

前三者难以扩展，每个路径需要一个寄存器和比较器。

calendar queue 只能支持小规模的优先级数。

文中提出的方案支持任意数量的条目、任意数量的优先级以及任何数据类型。

## Overview

输出端：两个字段：priority、data

核心模块：

1.  data field storage module：接收输入端的新元素（通过 priority 与 data 字段与其他元素隔离开来，存储地址指针与优先级字段组成新的元素）
2.  enqueue-dequeue pipeline：将新元素完成入队操作，并将其插入到 max-heap module
3.  max-heap module：出队操作，将根节点的地址指针提供给 data field storage module，并将根节点出队（根节点出对导致的空缺将通过 enqueue-dequeue pipeline 传递到叶子节点）

可参数化：

1.  队列深度
2.  优先级数量
3.  数据类型

## POINTER ARRAY: ENQUEUE-DEQUEUE PIPELINE

1.  输入以 pipeline 的形式从根节点传递到叶节点
2.  输出根节点，自下而上填补空位
3.  出队、入队相互独立

### 数据结构：P

*   operation：当前的层次的操作（enqueue、dequeue、non-op、enqueue-dequeue）
*   priority
*   address：数据在 data field storage 中的存储位置
*   location：指向当前的层级的指针，指示出队或入队操作的路径（从根节点到叶节点）。以出队为例，location 指示了空位被传递到叶节点的路径

pipeline enqueue/dequeue：每个操作之间的指令延迟为 2。

### Pipeline Stage of EDP

enqueue operation：

1.  将 p.priority 与 p.location 指向的节点的优先级相比较，若 p.location 指向的节点为空，则跳至 step2；若 p.location 指向的节点比 p 中的优先级大，则跳至 step3；若 p.location 指向的节点比 p 中的优先级小，则跳至 step4；
2.  将 P 放至 p.location 指向的节点，将 B.capacity - 1
3.  将 p 传递至下一层，并更新 p 的 location
4.  将 P 与 max-heap 模块中存储的节点交换，利用其原有左右子节点的容量将新的 P 向下传递到下一层。
