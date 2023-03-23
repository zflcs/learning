
#### 内核进程控制块数据结构
```rust
pub struct ProcessControlBlockInner {
    pub is_zombie: bool,
    pub is_sstatus_uie: bool,
    pub memory_set: MemorySet,
    pub user_trap_info: Option<UserTrapInfo>,
    pub parent: Option<Weak<ProcessControlBlock>>,
    pub children: Vec<Arc<ProcessControlBlock>>,
    pub exit_code: i32,
    pub fd_table: Vec<Option<Arc<dyn File + Send + Sync>>>,
    pub tasks: Vec<Option<Arc<TaskControlBlock>>>,
    pub task_res_allocator: RecycleAllocator,
    pub user_trap_handler_tid: usize,
    pub user_trap_handler_task: Option<Arc<TaskControlBlock>>,
    pub mutex_list: Vec<Option<Arc<dyn SimpleMutex>>>,
    pub condvar_list: Vec<Option<Arc<Condvar>>>,
}
```
以进程为单位注册用户态中断

#### 用户进程注册用户态中断

`init_user_trap` 函数注册，先注册一个线程，再执行 `sys_init_user_trap` 系统调用
```rust
pub fn init_user_trap() -> isize {
    let tid = thread_create(user_interrupt_handler as usize, 0);
    let ans = sys_init_user_trap(tid as usize);
    ans
}
```

内核 `sys_init_user_trap` 系统调用处理，向进程注册用户态中断处理线程
```rust
pub fn sys_init_user_trap(user_trap_handler_tid: usize) -> isize {
    current_process().unwrap().set_user_trap_handler_tid(user_trap_handler_tid);
    match current_process()
        .unwrap()
        .acquire_inner_lock()
        .init_user_trap()
    {
        Ok(addr) => {
            return addr;
        }
        Err(errno) => {
            return errno;
        }
    }
    -1
}
```
1. 先设置 PCB 中的字段
2. PCB 初始化用户态中断，通过 mmap 映射一个页面，在 `user_trap_info` 指向映射的页面，使用 `spsc` 队列存放用户态中断的消息，打开 uie 位，允许中断使能
```rust
    pub fn init_user_trap(&mut self) -> Result<isize, isize> {
        use riscv::register::sstatus;
        if self.user_trap_info.is_none() {
            // R | W
            if self.mmap(USER_TRAP_BUFFER, PAGE_SIZE, 0b11).is_ok() {
                let phys_addr =
                    translate_writable_va(self.get_user_token(), USER_TRAP_BUFFER).unwrap();
                self.user_trap_info = Some(UserTrapInfo {
                    user_trap_buffer_ppn: PhysPageNum::from(PhysAddr::from(phys_addr)),
                    devices: Vec::new(),
                });
                let trap_queue = self.user_trap_info.as_mut().unwrap().get_trap_queue_mut();
                *trap_queue = UserTrapQueue::new();
                unsafe { sstatus::set_uie(); }
                self.is_sstatus_uie = true;
                return Ok(USER_TRAP_BUFFER as isize);
            } else {
                warn!("[init user trap] mmap failed!");
            }
        } else {
            warn!("[init user trap] self user trap info is not None!");
        }
        Err(-1)
    }
```

#### 中断处理线程
之前创建的普通线程被调度执行，打开软中断和时钟中断，执行 `hang` 系统调用
```rust
fn user_interrupt_handler() {
    extern "C" { fn __alltraps_u(); }
    unsafe {
        utvec::write(__alltraps_u as usize, TrapMode::Direct);
        uie::set_usoft();
        uie::set_utimer();
    }
    loop { hang(); }
}
```

内核 `hang` 系统调用，判断是否注册了用户态中断，由于创建线程和初始化 `user_trap_info` 是两阶段工作，因此前一个判断有必要。如果已经初始化了 `user_trap_info` ，并且消息队列为空，则将这个普通线程注册成中断处理线程，并将这个线程阻塞，如果不满足条件，则这个线程让权
```rust
pub fn sys_hang() -> isize {
    let task = current_task().unwrap();
    let process = task.process.upgrade().unwrap();
    let mut process_inner = process.acquire_inner_lock();
    let pid = process.pid.0;
    if process_inner.user_trap_info.is_some() && process_inner.user_trap_info.as_ref().unwrap().get_trap_queue().is_empty() {
        process_inner.user_trap_handler_task = Some(task);
        drop(process_inner);
        drop(process);
        remove_uintr_task(pid);
        block_current_and_run_next();
    } else {
        drop(process_inner);
        drop(process);
        suspend_current_and_run_next();
    }
    0
}
```

#### 内核向用户进程发送用户态中断




#### 外设发送用户态中断
