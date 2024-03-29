## Kotlin Coroutines: Design and Implementation

### 异步编程的问题：

- 函数着色：异步和同步代码之间的区分；同步代码中需要过渡代码使用异步代码
- 异步计算比正常的函数调用，带来了额外的开销
- 针对异步代码的错误处理更加复杂，异步的 debug 也很困难

有栈协程可以使用统一颜色的代码；而无栈协程则会根据暂停的要求需要两种颜色的代码，也可能是完全异步的代码

### Kotlin 协程

- 返回时将正常结果返回
- 暂停时，返回 suspend 标记，**不允许用户手动返回 suspend 标记**
- 状态机

### Current Limitations

- 颜色透明
- 平台互操作性：其他语言调用 Kotlin
- 协程与面向对象编程