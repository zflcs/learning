---
tags: []
parent: 'SystemVerilog for Verification: A Guide to Learning the Testbench Language Features'
collections:
    - HDL学习
version: 8131
libraryID: 1
itemKey: 2ILURFV4

---
# SystemVerilog 学习笔记

## Data Types

### Built-in Data Types

*   无符号单个或多比特变量：reg\[7:0] m
*   32位有符号变量：integer
*   64位无符号变量：time
*   浮点数：real

#### The logic type

在原本的 reg 类型基础上增加了一些额外的功能，可以由连续赋值、门和模块驱动。并且用 **logic **来声明类型。

#### Two-state types

bit、byte、shortint、int、longint

#### Fixed-Size Arrays

类似与 C 语言数组的语法，使用 for 或 foreach 遍历。

在变量名之前的为 packed，在变量名之后的为 unpacked。

#### Dynamic Arrays

使用 \[] 来声明空的动态数组，new\[num] 进行初始化。

#### Queues

#### Associative Arrays

#### Linked Lists

#### Array Methods

#### Creating New Types with typedef

typedef
