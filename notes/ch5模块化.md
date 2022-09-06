需要资源分配模块

PCB：依赖地址空间模块、资源分配模块、context 模块

TCB：依赖资源分配模块、context 模块

协程 future：context



在 loop 循环中，每次 MANAGER 会取出一个任务，然后可以调用 execute 方法，执行代码（进程、线程、协程），然后来了中断之后回到内核 loop 循环中，然后执行系统调用或其他异常处理，例如，执行 sys_fork 系统调用，会执行相应的代码，创建新的任务并添加进 MANAGER 中，大致的流程如下，其中会涉及到 suspend、exit 等，依赖 syscall 模块。



管理器：（进程、线程、协程）单例模式，依赖 context 模块

```rust
/// 进程、线程、协程都必须实现这个 trait
pub trait Execute {
    fn execute();
}

/// 进程管理器能够管理的是实现了 executor 这个特性的任务
pub trait Manager: Sync {
	fn fetch(&self) -> &Arc<dyn Execute + Sync + Send>;
    fn add(&self, task: &Arc<dyn Execute + Sync + Send>);
}

/// 任务管理器
static MANAGER: Once<&'static dyn Manager> = Once::new();

/// 设置全局的管理器对象
pub fn init_manager(manager: &'static dyn Manager) {
    MANAGER.call_once(|| manager);
}
```



任务管理：依赖 syscall 模块

fork + exec + spawn + waitpid：进程

yield + exit：进程、线程、协程

```rust
pub fn fork();
pub fn exec();
pub fn spawn();
pub fn yield();
pub fn exit();
pub fn waitpid();
```



##### 面临的问题：

MANAGER 中取出的不知道是进程、线程还是协程

这时候执行系统调用处理时，内核不知道



