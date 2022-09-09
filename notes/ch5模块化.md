#### 第五章模块化

##### 总体思路：

在 tasks 中保存所有的进程对象，并且建立 task 与 id 的映射关系

调度通过维护 task_queue，将 id 插入到 task_queue 中，表示进程需要调度，如果 id 不在 task_queue 中，则不需要进行调度

这样的好处：不需要额外建立 PID2PCB 的结构，然后进程之间的父子关系可以只通过 id 来维护

然后进程内封装了地址空间和上下文，tutorial 中的很多接口也都不需要了

##### 具体的接口变化

不再需要 PROCESS 以及 PID2PCB 这两个全局变量

```rust
// 这些接口实现的功能直接由 MANAGER 提供了
pub fn pid2process(pid: usize) -> Option<Arc<ProcessControlBlock>>
pub fn insert_into_pid2process(pid: usize, process: Arc<ProcessControlBlock>)

// 这些接口在目前可以直接根据当前进程的 id，获取到对应的 PCB，然后直接获取到地址空间以及上下文
// 可以直接调用地址空间中的 translate 完成地址转换
pub fn current_user_token() -> usize;
pub fn current_trap_cx() -> &'static mut TrapContext;
pub fn current_trap_cx_user_va() -> usize;
```

pub fn suspend_current_and_run_next()：进入内核，将进程 id 通过 MANAGER.add 方法添加回调度队列里面

pub fn block_current_and_run_next()：不需要将进程 id 添加到 task_queue 中，只需要将 id 添加到 Mutex 或者 Semaphore 的阻塞队列中，唤醒操作也是将 id 添加到 task_queue 中

pub fn exit_current_and_run_next(exit_code: i32)：首先根据 id 维护进程之间的关系，然后将进程对象从 BTreeMap 中删除即可

根据进程的 id 来维护，第八章信号那一部分的接口，我觉得是没有问题的



之前讨论的 syscall 的问题，也可以直接通过 MANAGER 提供的 current 接口来解决

```rust
/// 任务管理器
/// `tasks` 中保存所有的任务实体
/// `task_queue` 负责进行调度，任务需要调度，则任务的 id 会在 task_queue 中
/// 从中取出任务，并不会删除任务，之后需要调度则需要将 id 重新插回 `task_queue` 中
/// 只能通过 `del` 才可以删除任务的实体
pub struct TaskManager<T> {
    /// 任务
    tasks: BTreeMap<usize, T>,
    /// 任务队列
    task_queue: Vec<usize>,
    /// 当前正在执行的任务 id
    current: usize,
}

impl<T> TaskManager<T> {
    /// 新建任务管理器
    pub const fn new() -> Self;
    /// 插入一个新任务
    pub fn insert(&mut self, id: usize, task: T);
    /// 根据 id 获取对应的任务
    pub fn get_task(&mut self, id: usize) -> Option<&mut T>;
    /// 将没有执行完的任务重新插回调度队列中
    pub fn add(&mut self, id: usize);
    /// 删除任务实体
    pub fn del(&mut self, id: usize);
    /// 取出任务
    pub fn fetch(&mut self) -> Option<&mut T>;
    /// 当前正在执行的任务
    pub fn current(&mut self) -> Option<&mut T>;
}

```



##### 上周完成的部分内容：

- 和杨德睿讨论之后，在地址空间中增加了段的管理，利用 mapper 以及 visitor 实现了地址空间复制的接口

- 开始着手写 fork、exec等系统调用的实现





一些其他的想法：对于 Mutex 以及 Semaphore 是否也可通过这种方式？？？ 使得 MANAGER 不只是对于任务，对于其他的资源也可以实现管理？？？



