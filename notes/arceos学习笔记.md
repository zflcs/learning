
# Arceos 学习笔记

## 启动流程

### modules/axhal/src/platform/***/boot.rs

1. 在 _start 的汇编代码中，设置好 boot_stack 后，初始化 boot_page_table（设置好对等映射 0x8000_0000..0xc000_0000 和高位虚拟地址映射 0xffff_ffc0_8000_0000..0xffff_ffc0_c000_0000），开启 MMU
2. 将使用的对等映射的寄存器复制到高位虚拟地址
3. 进入 rust_entry 函数

### modules/axhal/src/platform/***/mod.rs

1. 调用 clear_bss 将 bss 段清空
2. 调用 init_primary 初始化 CPU（def_percpu 宏定义了一些对全局变量的操作）
3. 设置 trap 的入口
4. 进入外部定义的 rust_main 函数中

### modules/axruntime/src/lib.rs

1. 打印一些基础信息
2. 初始化 log
3. 打印内存区域信息
4. 初始化堆分配器
5. 初始化内核页表
6. 初始化平台设置（irq、time 初始化）
7. 如果开启了 multitask feature，则初始化调度器
8. 初始化设备驱动，以及相应的内核模块（文件系统、网络协议栈、显卡）
9. 如果开启了 smp feature，则启动其他核
10. 如果开启了 irq，则初始化中断子系统
11. 如果开启了 tls 且没有开启 multitask，则初始化 tls
12. 等初始化完成后，进入到 main 函数中

至此，已经进入了自定义的代码

## 多任务调度

1. 调用 init_scheduler/init_scheduler_secondary 初始化调度器
2. 主核直接调用 main 函数，而其他的核调用 run_idle

### axtask/src/api.rs

api_s.rs 中定义了单任务 yield、sleep 等函数

init_scheduler/init_scheduler_secondary 函数

1. 初始化 run_queue
2. 如果开启了 irq feature，则初始化 timers

run_idle 函数

1. 循环调用 yield_now
2. 如果开启了 irq feature，则调用 wait_for_irqs

### axtask/src/run_queue.rs

初始化 IDLE_TASK

这里会添加 main_task，但只是放了一个空的 task 结构在队列中，不与实际的 main 函数绑定，在开启 multitask feature 时，main 函数执行完毕后调用 exit，把 main_task 对应的结构从 run_queue 中删除

### 任务切换

没有 preempt：通过 yield、exit、block、sleep 等接口让出 CPU
开启 preempt：暂时先不考虑

yield、exit、block、sleep 等函数为同步的线程代码与异步的协程代码之间的边界，所以 LX 的做法是正确的，原本设想的方式在 arceos 中并不适用

对于调度的部分可以按照上述的接口进行修改，但对于直接使用函数调用实现的系统调用，这些部分是难以修改的。

在 axlibc 中的接口最终调用 arceos_posix_api 中的实现，因此，只需要在 api 模块中增加转化成异步代码的接口（准确的来说是在 utils.rs 目录中增加一个宏）

在这个宏里，需要做的事情是，对于那些同步的函数，还是采用原来的方式，对于那些可能会阻塞的接口，需要转化成异步的 future

在 modules/axnet/src/smoltcp_impl.rs 中增加了一些接口，对于 Err(AxError::WouldBlock) 会直接调用 yield_now 的接口，然后根据线程的上下文切换，可以保证下一次可以在这里运行，手动实现了切换。

但目前全为同步的接口，对于异步改造，需要设计出一个合理的接口，与上述的宏结合起来。这就是同步和异步的适配层。

### GC

需要通过 GC 回收退出的线程

## 协程封装进程

进程执行
1. 一种角度是主动从内核返回到用户态，这样返回到用户态的那段上下文保存和恢复被理解为入口，进入内核态的上下文保存和恢复时出口；
2. 另一种角度是假设线程原本就是在执行的，进程陷入内核那段上下文保存和恢复被视为入口，回到用户态的上下文保存和恢复被视为出口；

在 arceos 中，第二种思路是更为合适的，这种方式在每次通过系统调用进入内核时，都相当于会重启一个协程执行器，与这个系统调用相关的协程都由这个执行器管理。但是其他的与这次系统调用无关的协程应该怎么执行。

一种方案：

Task 的执行通过 TaskInner 函数进行，yield、exit、block、sleep 等函数最终让控制流回到 TaskInner 的 poll 函数，这样把整个 task 封装成一个协程，当通过 yield 回到 poll 函数之后，可以执行其他的协程

新增一个 get_cx 接口，获取当前的协程的 cx，这样即可在 block 时向中断控制器中注册阻塞的协程

## Run redis on arceos

make PLATFORM=riscv64-axu15eg A=apps/c/redis FEATURES=driver-axi-eth,driver-ramdisk LOG=info SMP=4

## iperf

需要在 feature.txt 中声明使用 multitask，才能使用 sched_atsintc 调度器，但还是无法，但 iperf 一旦建立连接后，就不会 yield，ATSINTC 无法使用

## redis

redis 使用了 epoll，目前的问题是使用了 atsintc 的调度器在开发板上无法正常建立连接，问题还不知道出在哪里

## 结论

目前利用 ATSINTC 的机制，在开发板上运行 arceos 的 redis 和 iperf 在短时间内是达不到目的的
