## A Survey of Asynchronous Programming Using Coroutines in the Internet of Things and Embedded Systems

### Introduction

- 异步编程困难，应用程序初始化的逻辑和外部事件响应时的逻辑是分开的（没有语言的支持）
  - 解决方案：语言对协程的支持，C#、js、python 使用 await 等待外部事件，因此使得异步编程像同步编程
- 嵌入式系统大多用 c、c++ 编程，缺少 await 的支持，c++ 的协程最初实现时没有考虑到嵌入式设备资源有限的情况

### Background

#### Async/Await Pattern

- c++ co_await、co_yield、co_return
- 优点：编程模式、避免了显式使用全局变量

#### Coroutines

一种方式为暂停时，存储局部变量的状态。另一种方式则是 Protothreads：协程中所有的变量都只能静态分配，减少了上下文开销，内存使用可预测，协程不可重入，程序员显式管理协程，更容易产生 bug

协程还可以更具有栈和无栈进行划分。有栈协程拥有自己的栈，不与调用者共享，因此可以在栈上存储局部变量；无栈协程暂停时需要从栈中弹出状态，需要通过其他机制保存状态。无栈协程不能在调用的子函数上暂停。

#### Previous Coroutine Implementations for Constrained Platforms

早期使用 C 中的宏来实现协程相关的特性，Duff 的设备利用了 C 语言 case 可以没有 break 的特性

Protothreads 存在两个缺点：不能在程序中安全的使用 switch 语句；不能管理局部变量，状态必须使用全局变量

### RESULTS

#### Use Cases

- 并发
- 网络通信
- 传感器读数
- 数据流

#### 状态保存方式

- Heap
- Stack
- Static/Global

C++ 中任何协程的实现都必须支持动态内存分配；受限制的设备这种特殊场景要求可以根据开发人员的选择使用不同的策略。

在受限的条件下，无栈协程更好

