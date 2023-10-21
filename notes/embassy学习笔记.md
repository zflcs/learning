### Embassy 学习笔记

使用 extern `Rust` 来定义接口，避免 trait 动态分发

针对小于 1us 的定时，采用 block 的方式，其他采用 await 方式

在 main 函数中，指定 spawner 用于创建其他任务，指定 peripherals，在其中配置需要使用的外设。

Executor 对于 pending 状态的任务，会将其插入到就绪队列队尾

可以使用 [static-cell](https://github.com/embassy-rs/static-cell) 结构保存 'static 生命周期的 Executor

在不支持原子操作的设备上，可以使用 [critical-section](https://github.com/rust-embedded/critical-section) 

pender 函数与数据结构的具体用途：用于通知存在协程需要尽快执行，在非中断模式下，还用于临界区的判断

spawner 创建 task 后，必须 spawn，需要 SpawnToken

为什么需要在 smoltcp 定义的 trait 中增加 context 参数：直接从 cx 参数得到对应的 waker，从而可以避免查找任务的时间，可以快速响应，在推动协程执行时，如果为 pending，可以直接使用 cx 得到的 waker，从而直接执行唤醒的操作，这中间省略了很多查找的过程，通过 will_wake 函数可以进行判断

通过创建多个 Executor 来支持不同的优先级，低优先级的 Executor 以线程模式工作（没有中断），使用 sev 来通知存在任务需要执行，使用 wfe 等待任务

非中断模式：需要等待低优先级任务执行完，才能执行高优先级任务

中断模式：高优先级任务可以直接抢占

Executor 中的原子链表一次性将所有任务都取出来，这个行为如何保证优先级高的任务进行抢占？只能依靠中断，正常的插入高优先级任务也不然是插入到另一个高优先级的 Executor 中



设备中断处理函数：on_interrupt 函数，对队列中的 item 依次进行查询操作，执行 waker 操作，唤醒协程



##### 定时器：

只能分配固定数量的 alarm

分配 alarm 后设置回调函数，并且传入 ctx 参数

在每个时刻只能设置一个 alarm

在发生时钟中断时，调用 on_interrupt 函数，调用 trigger_alarm 函数，执行预先定义的回调函数，回调函数中设置了 ctx，因此可以得到对应的 waker，执行唤醒过程



#### 整体流程

thread 模式，以 embassy/stm32f0/src/bin/hello.rs 测例示意：

#[embassy_executor::main] 宏指定了入口函数，并且初始化 Executor，默认是以 thread 模式工作，在没有任务时执行 wfe 指令进入低功耗模式，通过 sev 指令唤醒，这种模式没有中断，一旦发送了 sev 指令，即相当于产生了时钟中断，但不会进行抢占，继续从 await 之后执行

```rust
#[embassy_executor::main]
async fn main(_spawner: Spawner) -> ! {
    let _p = embassy_stm32::init(Default::default());
    loop {
        Timer::after_secs(1).await;
        info!("Hello");
    }
}
```



interrupt 模式，以 embassy/stm32f0/src/bin/multiprio.rs 测例示意：

通过 #[entry] 宏指定入口函数，初始化三个优先级的 Executor（EXECUTOR_LOW、EXECUTOR_MED、EXECUTOR_HIGH），设置对应的中断处理函数，并且得到各自 Executor 对应的 Spawner（只能向对应的 Executor 中添加任务）；EXECUTOR_LOW 以 thread 模式工作，不需要指定中断处理函数，EXECUTOR_MED、EXECUTOR_HIGH 以 interrupt 模式工作，指定中断处理函数为 interrupt::USART2 和 interrupt::USART1，分别处理不同优先级的中断

```rust
#[entry]
fn main() -> ! {
    // Initialize and create handle for devicer peripherals
    let _p = embassy_stm32::init(Default::default());

    // High-priority executor: USART1, priority level 6
    interrupt::USART1.set_priority(Priority::P6);
    let spawner = EXECUTOR_HIGH.start(interrupt::USART1);
    unwrap!(spawner.spawn(run_high()));

    // Medium-priority executor: USART2, priority level 7
    interrupt::USART2.set_priority(Priority::P7);
    let spawner = EXECUTOR_MED.start(interrupt::USART2);
    unwrap!(spawner.spawn(run_med()));

    // Low priority executor: runs in thread mode, using WFE/SEV
    let executor = EXECUTOR_LOW.init(Executor::new());
    executor.run(|spawner| {
        unwrap!(spawner.spawn(run_low()));
    });
}
```

#[interrupt] 宏，设置针对不同中断的中断处理函数

```rust
#[interrupt]
unsafe fn USART1() {
    EXECUTOR_HIGH.on_interrupt()
}

#[interrupt]
unsafe fn USART2() {
    EXECUTOR_MED.on_interrupt()
}
```

#[embassy_executor::task] 生成任务元信息，包括 future 以及其他元信息

```rust
#[embassy_executor::task]
async fn run_high() {
    loop {
        // info!("        [high] tick!");
        Timer::after_ticks(27374).await;
    }
}

#[embassy_executor::task]
async fn run_med() {
    loop {
        let start = Instant::now();
        info!("    [med] Starting long computation");

        // Spin-wait to simulate a long CPU computation
        cortex_m::asm::delay(8_000_000); // ~1 second

        let end = Instant::now();
        let ms = end.duration_since(start).as_ticks() / 33;
        info!("    [med] done in {} ms", ms);

        Timer::after_ticks(23421).await;
    }
}

#[embassy_executor::task]
async fn run_low() {
    loop {
        let start = Instant::now();
        info!("[low] Starting long computation");

        // Spin-wait to simulate a long CPU computation
        cortex_m::asm::delay(16_000_000); // ~2 seconds

        let end = Instant::now();
        let ms = end.duration_since(start).as_ticks() / 33;
        info!("[low] done in {} ms", ms);

        Timer::after_ticks(32983).await;
    }
}

//!     [med] Starting long computation
//!     [med] done in 992 ms
//!         [high] tick!
//! [low] Starting long computation
//!     [med] Starting long computation
//!         [high] tick!
//!         [high] tick!
//!     [med] done in 993 ms
//!     [med] Starting long computation
//!         [high] tick!
//!         [high] tick!
//!     [med] done in 993 ms
//! [low] done in 3972 ms
//!     [med] Starting long computation
//!         [high] tick!
//!         [high] tick!
//!     [med] done in 993 ms
```

以 thread 模式工作的 EXECUTOR_LOW 运行，目前只有一个任务 run_low，run_low 任务被 poll 一次之后，在 await 处让权，此时执行 wfe 指令等待 sev 指令

```rust
pub fn run(&'static mut self, init: impl FnOnce(Spawner)) -> ! {
    init(self.inner.spawner());

    loop {
        unsafe {
            self.inner.poll();
            asm!("wfe");
        };
    }
}
```

以 interrupt 模式工作的 EXECUTOR_MED、EXECUTOR_HIGH 在发生中断后，调用对应的 on_intertupt 函数

```rust
/// You MUST call this from the interrupt handler, and from nowhere else.
pub unsafe fn on_interrupt(&'static self) {
    let executor = unsafe { (&*self.executor.get()).assume_init_ref() };
    executor.poll();
}
```

poll 函数中将 timer_queue 中所有 expires_at 位于当前时刻之前的任务，通过 wake_task_no_pend 函数唤醒，wake_task_no_pend 将会更新任务状态，并且添加进就绪队列中，再从就绪队列中将任务取出，执行各自的 poll_fn 函数

```rust
/// Same as [`Executor::poll`], plus you must only call this on the thread this executor was created.
pub(crate) unsafe fn poll(&'static self) {
    #[cfg(feature = "integrated-timers")]
    driver::set_alarm_callback(self.alarm, Self::alarm_callback, self as *const _ as *mut ());

    #[allow(clippy::never_loop)]
    loop {
        #[cfg(feature = "integrated-timers")]
        self.timer_queue.dequeue_expired(Instant::now(), wake_task_no_pend);

        self.run_queue.dequeue_all(|p| {
            let task = p.header();

            #[cfg(feature = "integrated-timers")]
            task.expires_at.set(Instant::MAX);

            let state = task.state.fetch_and(!STATE_RUN_QUEUED, Ordering::AcqRel);
            if state & STATE_SPAWNED == 0 {
                // If task is not running, ignore it. This can happen in the following scenario:
                //   - Task gets dequeued, poll starts
                //   - While task is being polled, it gets woken. It gets placed in the queue.
                //   - Task poll finishes, returning done=true
                //   - RUNNING bit is cleared, but the task is already in the queue.
                return;
            }

            #[cfg(feature = "rtos-trace")]
            trace::task_exec_begin(p.as_ptr() as u32);

            // Run the task
            task.poll_fn.get().unwrap_unchecked()(p);

            #[cfg(feature = "rtos-trace")]
            trace::task_exec_end();

            // Enqueue or update into timer_queue
            #[cfg(feature = "integrated-timers")]
            self.timer_queue.update(p);
        });

        #[cfg(feature = "integrated-timers")]
        {
            // If this is already in the past, set_alarm might return false
            // In that case do another poll loop iteration.
            let next_expiration = self.timer_queue.next_expiration();
            if driver::set_alarm(self.alarm, next_expiration.as_ticks()) {
                break;
            }
        }

        #[cfg(not(feature = "integrated-timers"))]
        {
            break;
        }
    }

    #[cfg(feature = "rtos-trace")]
    trace::system_idle();
}
```



```rust
/// Wake a task by `TaskRef` without calling pend.
///
/// You can obtain a `TaskRef` from a `Waker` using [`task_from_waker`].
pub fn wake_task_no_pend(task: TaskRef) {
    let header = task.header();

    let res = header.state.fetch_update(Ordering::SeqCst, Ordering::SeqCst, |state| {
        // If already scheduled, or if not started,
        if (state & STATE_RUN_QUEUED != 0) || (state & STATE_SPAWNED == 0) {
            None
        } else {
            // Mark it as scheduled
            Some(state | STATE_RUN_QUEUED)
        }
    });

    if res.is_ok() {
        // We have just marked the task as scheduled, so enqueue it.
        unsafe {
            let executor = header.executor.get().unwrap_unchecked();
            executor.run_queue.enqueue(task);
        }
    }
}
```





task 数据结构

```rust
#[repr(C)]
pub struct TaskStorage<F: Future + 'static> {
    raw: TaskHeader,
    future: UninitCell<F>, // Valid if STATE_SPAWNED
}
```

raw 表示任务的元信息，我理解 embassy 应该是类似于网络协议一样的处理，任务的元信息用 TaskHeader 来命名，而实际的任务则是 future

```rust
/// Raw task header for use in task pointers.
pub(crate) struct TaskHeader {
    pub(crate) state: AtomicU32,
    pub(crate) run_queue_item: RunQueueItem,
    pub(crate) executor: SyncUnsafeCell<Option<&'static SyncExecutor>>,
    poll_fn: SyncUnsafeCell<Option<unsafe fn(TaskRef)>>,

    #[cfg(feature = "integrated-timers")]
    pub(crate) expires_at: SyncUnsafeCell<Instant>,
    #[cfg(feature = "integrated-timers")]
    pub(crate) timer_queue_item: timer_queue::TimerQueueItem,
}
```

任务元信息：

state 任务状态

run_queue_item 下一个任务的 TaskHeader，链表

executor 当前任务在对应的 executor 上运行

poll_fn：调用 Future::poll 外层的函数，就是任务 async 函数与普通函数的边界

expires_at：用于定时器

timer_queue_item：链表



```rust
/// This is essentially a `&'static TaskStorage<F>` where the type of the future has been erased.
#[derive(Clone, Copy)]
pub struct TaskRef {
    ptr: NonNull<TaskHeader>,
}
```

TaskRef：指针，在 embassy 自定义的 waker 中，根据这个指针，找到对应的 Waker



#### 借鉴之处

我们的设计最大的问题在于将数据结构之间的关系没有联系起来，完全是独立的操作，而 embassy 的数据结构之间有明显的联系

- 在 driver 的接口中增加 context 参数，利用 context 找到 waker，进行唤醒
- TaskRef 与 TaskHeader、Waker 之间的关系，TaskRef 与 RunQueue、TimerQueue 之间的关系
- TaskHeader 与 Executor
- Spawner 与 Executor



针对共享调度器第一个实验结果，不考虑优先级，使用 embassy 里面定义的数据结构会更好，但是它的 executor 定义的是批处理的模式，对于优先级会有一些影响

