## Revisiting Coroutines

协程的基本特征：

- 协程的局部数据在连续的调用之间持久存在
- 控制离开时，协程被暂停，之后重新进入协程时，继续执行

构造协程面临的问题：（除了保存状态，还有根据三个主要问题来区分协程的类型）

- 控制传递机制（对称/非对称）
- 在语言中是作为一级对象提供，由程序员自由操作，还是作为约束的构造提供
- 协程是否有栈，能否从嵌套的调用中暂停执行

协程缺乏精确的定义，由于 first-class 的出现，因此对于协程的研究主要是通用控制抽象，之后的语言不再提供 first-class。另一个原因是主流环境是多线程编程。

非对称协程更加易于管理和理解，并且支持更加结构化的应用程序。

### A CLASSIFICATION OF COROUTINES

#### Control Transfer Mechanism

- 对称协程：单一控制转移，允许协程之间显式传递控制权，需要程序员花更多的精力来理解控制转移
- 非对称协程：提供两种控制转移操作，一种用于调用协程、另一种用于暂停协程，将控制权返回给协程调用程序

#### First-Class versus Constrained Coroutines

- 任何地方都可以使用
- 只对于特定的数据结构内部提供协程的工具

#### Stackfulness

- Python 无栈协程：只能在创建协程的同一个栈帧中暂停
- 无栈协程不支持多任务环境：应该指的是状态的保存，下一次执行需要恢复状态，重新构造

### FULL  ASYMMETRIC  COROUTINES

#### Coroutine Operators

create/resume/yield：resume 和 yield 可以传递参数

### Cooperative Multitasking

协程提供了一种本质上是协作的并发模型：协程必须自愿释放控制以允许其他协程继续。对于共享资源的访问时简单的任务，对同步机制的需求被最小化。

尽管公平性问题不大，但是仍然是一个难题。应该指的是相互独立的协程之间的公平性。

### Symmetric Coroutines

#### Operator

create/transfer：transfer 带有参数，将控制权传递到参数指向的协程