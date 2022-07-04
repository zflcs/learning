### AsyncOS 异步操作系统

#### 多核的相关设置

1. 启动

    在 Makefile 启动 qemu 中添加 -smp n 来启动 n 个核

    ```makefile
    qemu-system-riscv64 \
    		-machine virt \
    		-m 128 \
    		-nographic \
    		-bios $(BOOTLOADER) \
    		-smp 4
    ```

    rustsbi 设置了内核的入口地址为 0x8020_0000，每个核的栈空间为 4KB，最多 8 个核

    ```rust
    mod constants {
        /// 特权软件入口。
        pub(crate) const SUPERVISOR_ENTRY: usize = 0x8020_0000;
        /// 每个核设置 16KiB 栈空间。
        pub(crate) const LEN_STACK_PER_HART: usize = 16 * 1024;
        /// qemu-virt 最多 8 核。
        pub(crate) const NUM_HART_MAX: usize = 8;
        /// SBI 软件全部栈空间容量。
        pub(crate) const LEN_STACK_SBI: usize = LEN_STACK_PER_HART * NUM_HART_MAX;
    }
    ```

    在内核的入口处先设置 tp 寄存器、各个核使用的栈，不能直接使 sp = boot_stack_top，而是根据硬件线程 hart_id 来分配空间

    ```assembly
        .section .text.entry
        .globl _start
    _start:
        # a0: hart id
        mv tp, a0
        la sp, boot_stack
        # li t1, 4096 * 16 # t1 = 4096 * 16 64KB
        addi t0, a0, 1  # t0 = a0 + 1 hartid+1
        slli t0, t0, 16 # 64K * (hartid + 1)
        add sp, sp, t0  # sp = sp + t0
        call rust_main
    
        .section .bss.stack
        .globl boot_stack
    boot_stack:
        .space 4096 * 16 * 4
        .globl boot_stack_top
    boot_stack_top:
    ```

    在内核 ·`rust_main` 函数，需要根据不同的 `hart_id` 来完成不同的工作，通过一个 `AtomicBool` 控制各个核之间的同步，当主核没有完成相关的初始化工作时，若其他核先启动，则会一直自旋，直到主核完成了初始化，同时需要对 `console.rs` 中的 `println!` 和` print!` 修改，将`stdout` 用 `Mutex` 包装起来，保证其执行的原子性，否则在打印时会比较混乱（暂时未找到 send_ipi 函数的相关文档，猜测是给其他核发送中断）

2. 各个核完成的任务

    主核：完成内存管理初始化、终端初始化、文件系统初始化等工作，之后会执行 `basic_rt::thread::init_cpu_test()`，创建回调队列、RRS调度器、线程池、idle主线程，并且在线程池中新增 `thread_main_ex` 线程，所有的线程都是在主核的线程池中

    其余核：执行 `cpu_run()`，通过函数调用的方式来启动协程执行器，然后 EXCUTOR 不断地查询 future 是否有进展，如果有进展，则将其从自己的任务（协程）队列中删除此任务，这个核只负责这个任务

3. 多个核之间的同步互斥

    只有两个核，一个核复制执行协程，另一个核只负责检测协程是否有进展（但是qemu 启动却用到了 4 个核）

4. 开启多核之后的变化

    - 需要维护 CPU_NUM 个 processor，每个 processor 上维护一个当前正在运行的线程
    - 线程池：封装了一个调度器，以及阻塞的线程队列
    - 调度器：每次从阻塞的线程队列中找到优先级最高的线程

5. 学长的工作的目标：让内核能够以某种方式参与协程的调度，例如，在有更高优先级的事件发生时，可以让协程让出 CPU，通过位图的方式来管理线程、协程的优先级，bitmap 并没有直接放在线程 TCB 中，而是直接放在物理内存中的固定位置，对于用户进程的 bitmap，则是预先在内存中申请 MAX_USER 个 bitmap，在运行时再分配给用户进程

    

    




在一个单独的核上处理协程，程序的地址是 0x87000000，没有其他的跳板页之类的机制

如果这个核发生了 panic，则是直接 loop{}，不进行其他额外的处理

##### init.rs：

##### cbq.rs：回调队列

##### task/bitmap.rs：

协程：没有返回值的函数调用，在这里是实现了 Future 特性的任何数据结构

```rust
pub future: Mutex<Pin<Box<dyn Future<Output=()> + 'static + Send + Sync>>>,
```











