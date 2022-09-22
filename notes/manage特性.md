#### Manage Trait

最初是因为任务管理调度需要模块化，因此开始抽象出来调度器

最开始调度器只有一个 fetch 和 add 接口，分别负责取出任务和添加任务，到后来慢慢的发现并不全面，一直发展到当前的版本，共 6 个接口

```rust
impl<T, I: Copy + Ord> TaskManager<T, I> {
    /// 新建任务管理器
    pub const fn new() -> Self;
    /// 插入一个新任务
    #[inline]
    pub fn insert(&mut self, id: I, task: T);
    /// 根据 id 获取对应的任务
    #[inline]
    pub fn get_task(&mut self, id: I) -> Option<&mut T>;
    /// 将没有执行完的任务重新插回调度队列中
    #[inline]
    pub fn add(&mut self, id: I);
    /// 删除任务实体
    #[inline]
    pub fn del(&mut self, id: I);
    /// 取出任务
    #[inline]
    pub fn fetch(&mut self) -> Option<&mut T>;
    /// 获取当前正在执行的任务
    #[inline]
    pub fn current(&mut self) -> Option<&mut T>;
}
```

但是在抽象出来的时候，发现不只是进程、线程、协程可以这样管理，其他的资源也可以这样管理，通过 ID 来进行，组成一种层次化的结构

通过 ID 在单核上目前设想的还没有遇到问题，但是在多核上，似乎通过 ID 就不怎么可行了，例如从 Mutex 的阻塞队列中取出一个进程 ID，但是在哪个核上的 MANAGER 找到对应的进程呢？？？取出了 ID 之后再插回到哪个 MANAGER 的队列中呢？？？

MANAGER 如果是所有核共用同一个？？？那 MANAGER 就会上锁，势必会影响性能；还是只能在每个核设置一个 MANAGER，每个核在执行的时候，可以获取自己的 hart_id，所以这个问题就解决了；资源目前看来也可以这样解决

尝试把 sync 抽出来模块化，但是需要依赖 suspend_current_and_run_next、block_current_and_run_next

这两个接口单纯靠 task-manage 不能实现，因此还需要将 processor 也抽象出来，由 processor 来提供这两个接口

在进程这个层面，task-manager 由 processor 负责

在线程这个层面，task-manager 由主线程负责

**刚刚看了廖东海的demo，提醒了我一下，可以再写一个 scheduler，实现不同的调度**

processor 接口

所有的进程转由 processor 来存放，manager 来实现 push、pop ID 这个功能

感觉类似于 批处理系统一样，但是不是顺序运行了

```rust

use super::TaskManager;

/// 处理器
pub struct Processor<T, I: Copy + Ord> {
    // 进程管理调度
    manager: TaskManager<T, I>,
    // 当前正在运行的进程 ID
    current: Option<I>,

}

impl <T, I: Copy + Ord> Processor<T, I> {
    pub fn new() -> Self;
    pub fn run_next(&mut self);
    pub fn make_current_suspend(&mut self);
    pub fn make_current_exited(&mut self);
}
```
