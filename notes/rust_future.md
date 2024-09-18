# 关于 Rust Future 的理解

协程顾名思义是以一种协作式的方式进行控制权的转移。Rust 语言的 Future 为非对称无栈协程（具体的定义可以看 Revisiting Coroutines 这篇论文）。

1. 非对称指的是任务将控制权转移到一个单独的调度器中（Rust Future 返回 Pending 后返回到最上层的函数中，取出下一个 Future 执行）；
2. 对称是指一个任务将控制权直接转移到下一个任务（Starry 是采取的对称的方式）。

关于 Rust 语言的几种 Future 实现方式：Compiler generated Future、User-implemented Future，定义来自于（论文 CAT: Context Aware Tracing for Rust Asynchronous Programs）。

Compiler generated Future 是内存布局不确定的，而 User-implemented Future 的内存布局是确定的。

## 状态机与迭代器的理解

1. 状态机的理解可以看这个[博客](https://wiki.brewlin.com/wiki/compiler/rust%E5%8D%8F%E7%A8%8B_%E8%B0%83%E5%BA%A6%E5%99%A8%E5%AE%9E%E7%8E%B0/#rust-%E7%8A%B6%E6%80%81%E6%9C%BA)。
2. Rust Future 与迭代器的关系可以看这个[博客](https://www.tisonkun.org/2023/11/06/trans-why-async-rust/#%E8%BF%AD%E4%BB%A3%E5%99%A8)。

编译器提供了 await 语法糖，将其转化成两个状态（Ready、Pending），因此执行单个顶层 Future 实际上是使用一个外部迭代器轮询内部的子 Future。

对于整个程序而言，实际上也是使用一个迭代器来构建高并发的程序。轮询的对象不再局限于单个顶层 Future（系统中的任务），而是这些顶层 Future 共同组成。因此需要由任务管理模块（Executor）来进行维护。

## cx、waker 与 Executor、回调的关系

cx 与 waker 的实现可以看 [embassy](https://github.com/embassy-rs/embassy/blob/main/embassy-executor/src/raw/waker.rs) 的实现。

cx 与 waker 实际上是回调的一种实现方式，但避免了回调地狱，在等待外部事件或其他的操作时，通过 waker.wake() 进行回调，唤醒任务。cx 与 waker 这两个结构是等价的，并且与 Executor 密切相关，在 Executor 中构造出来，在对单个顶层 Future 进行迭代时，将 cx 参数层层传递，一旦返回 Pending，则会注册 waker。

cx 中保存 waker 的指针，而 waker 中记录了一个类型擦除的指针，这个指针实际上是 Executor 中定义的 task 的指针。

## Everything is a Future

采用 Rust 协程来构建内核和用户态程序，自然想到了 `Everything is a Future` 的设计理念，但没有必要将所有的数据结构严格的按照 Future trait 的定义进行实现。（尝试过严格按照 Future trait 的定义进行实现，最后在实现过程中会导致把大量的时间花在了与编译器斗争的情况）。只需要按照协程的方式将原本的阻塞操作以非阻塞 + 注册回调的方式即可。以 async 实现 lock 的操作为例：

```rust
pub fn wait_until(
    &self, 
    cx: &mut Context<'_>, 
    condition: impl Fn() -> bool
) -> Poll<()> {
    let waker_node = Arc::new(WaitWakerNode::new(cx.waker().clone()));
    if condition() {
        self.queue.lock().remove(&waker_node);
        Poll::Ready(())
    } else {
        self.queue.lock().prepare_to_wait(waker_node);
        Poll::Pending
    }
}
```

这里没有实现 Future trait，但参数中增加了 cx，当条件未满足时，注册回调并且返回 Pending，这个结果会一直返回到顶层 Future。

在阅读了 [futures-io](https://github.com/rust-lang/futures-rs/blob/master/futures-io/src/lib.rs)、[embassy-net](https://github.com/embassy-rs/embassy/tree/main/embassy-net-driver) 的接口定义后，这些库中定义的接口都增加了 cx 这个参数。
