#### 抢占式的线程切换：

- 能够运行不可靠的用户态程序

- 为每个线程单独准备一个栈，避免了调用栈中的信息的保存与切换，只需要保存寄存器中的上下文
- 但是既是优点也是缺点，每个线程都必须要有自己的栈，内存开销增大，并且限制了系统的最大线程数
- 另一个缺点是 OS 必须保存完整的寄存器状态，即使任务只使用少量的寄存器

协作式线程切换

- 只能运行完成或者在协作的点来主动让出 CPU，例如等待 IO 时
- 协作式任务能够和异步操作结合起来，异步操作会返回一个 not ready 状态，然后执行 yield 来让出 CPU
- 保存状态：因为任务自己定义了可以暂停的点，因此不需要操作系统来保存状态，相反，协作式线程切换能够准确的保存恢复下次执行需要的各种信息，因此这会让其性能更好，例如进行了复杂的计算之后，只需要保存计算结果，而不需要保存中间的信息
- 语言支持的协作式任务能够在暂停之前保存需要的信息，因此多个任务可以共享同一个调用栈，减少内存，理论上能够创建任意数量的协作任务
- 协作式任务的缺点：一个非协作式的任务可能会永久运行，因此必须保证所有任务都是协作式的，但是让 OS 依赖于任务的协作性，这显然是不合常理的
- 但是良好的性能以及较小的内存开销，在一个程序内使用协作式任务显然是一个好的思路，尤其是和异步操作相结合时，



#### Future ：

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

then、map 等 combinator methods 会让编程变得很复杂，因为会发生所有权的转移以及生命周期不同等问题，而 async/await 则让代码看起来像同步的代码

状态机转换：并不是 wait 与 wait 之间可以暂停，而是整个 wait 是一个暂停点，在这个过程中，可以把 CPU 让出来

保存状态：不需要保存在 wait 时的所有状态，因为它知道下一个过程需要的所有数据，因此可以定制保存信息的数据结构

每个阶段都会根据 .await 前后所需要的信息来定制上下文数据结构，状态划分是根据 .await 来进行，在 .await 出时会完成一个状态转换的过程（*self = ......）

```rust
ExampleStateMachine::Start(state) => {
    // from body of `example`
    let foo_txt_future = async_read_file("foo.txt");
    // `.await` operation
    let state = WaitingOnFooTxtState {
        min_len: state.min_len,
        foo_txt_future,
    };
    *self = ExampleStateMachine::WaitingOnFooTxt(state);
}
```

完成状态转换之后，通过最外层的 loop 循环，就可以推进到另一个状态

```rust
impl Future for ExampleStateMachine {
    type Output = String; // return type of `example`

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        loop {
            match self { // TODO: handle pinning
                ExampleStateMachine::Start(state) => {…}
                ExampleStateMachine::WaitingOnFooTxt(state) => {…}
                ExampleStateMachine::WaitingOnBarTxt(state) => {…}
                ExampleStateMachine::End(state) => {…}
            }
        }
    }
}
```



#### 如何解决自引用结构移动的问题

1. 在移动之后，更新自引用结构中的指针，但是会对性能造成较大的影响，因为需要对所有的数据域进行检查
2. 不再存放指针，而是指针指向的数据在子引用结构中的偏移
3. 禁止移动自引用结构，不用再额外增加 runtime 开销，将处理的负担转移给写出自引用结构的程序员

> 虽然是禁止了，但是可以通过在堆上创建自引用结构，通过 Box 包装起来使用，尽管指针会改变，但是堆上分配的自引用结构的位置却不会移动，但是还是有可能会破坏这个自引用结构，例如 mm::replacce 会将原来的自引用结构移动到栈上去，返回一个 &mut 的数据结构，但是这个结构里面的 self_ptr 仍然指向原来的位置，因此仅仅在堆上分配仍然不能够保证安全的使用自引用结构
>
> 因为还存在上述问题，因此在 Box 外层再封装一个 Pin，需要在自引用结构内部增加一个标记类型，并且在申请数据实例时使用 Box::pin() 方法，虽然这样子禁止了获取 &mut 的可能，但是这样也禁止了申请实例之后的初始化，只能通过 unsafe、as_mut、get_unchecked_mut 来进行初始化，例如
>
> ```rust
> unsafe {
>     let mut_ref = Pin::as_mut(&mut heap_value);
>     Pin::get_unchecked_mut(mut_ref).self_ptr = ptr;
> }
> ```
>
> 虽然上述操作让我们能够安全的使用自引用结构，但是在堆上分配内存会对性能造成一定的影响，而且因此有探索出了在栈上分配的 Pin<&mut T> 方法，它只是暂时的借用包装的值，而不是真的拥有所有权，但是这需要程序员来确保其安全性，调用者必须将 Future 包装在首位来保证在内存中不会移动，因此还是使用上述方法，在堆上分配

#### 怎么来执行 Future？

之前的 async、await 等过程只是编译器创建好了状态机，但是实际上还是没有运行，只有在被 poll 的时候才会执行，必须要在某个地方调用。

- 可以手动的使用 loop 来循环的调用，但是这非常低效，并且在创建了大量的 Future 时不实用，因此需要定义一个全局的 executor 来负责 poll 所有的 Future，这样可以让 executor 来选择切换 Future，因此就可以执行异步操作，轮询则只能顺序执行，不能够灵活处理
- executor 会创建 waker，让 waker 在任务完成时能够释放信号，从而 executor 不必一直轮询



#### async/await 正是协作式多任务的一种实现方式



#### 在内核中添加对 async/await 的支持

1. 需要实现 executor 类，需要先实现 Task 类型，
2. 为 executor 准备好 waker，并且封装到上下文 context 中，为每个 future 提供一个 waker，能够在任务 Poll::ready() 时唤醒executor 来继续执行



#### 异步键盘输入

1. 

#### 存在的疑惑：

两个协程之间存在依赖关系，执行顺序是什么样的？？？当前的状态是 Poll::Pending 时，自己的状态没有改变，因此下一次 loop 还是会回到原来的分支，仍然是顺序执行的，并不是想象中的这个阻塞了，就可以运行下一个

状态的信息是怎么保存的呢，是由谁来保存呢？？？是由 OS 内核来完成这个过程吗？？？