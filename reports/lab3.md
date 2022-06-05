# 实验三报告

### 实践作业：增加系统调用获取任务信息

```
这个系统调用应该是给类似于 shell 的程序来使用，而不是给普通的应用程序来使用
思路：
	虽然提供了一个 TaskInfo 的数据结构，但是我觉得这些信息应该直接在 TaskControlBlock 中来记录，然后返回给应用程序的是 TaskInfo 这个数据结构
实现：
1. 在 syscall 模块中创建 SyscallInfo 数据结构，在 task 模块中创建 TaskInfo 数据结构，并实现相关的成员方法 new()
	struct SyscallInfo {
        id: usize,
        times: usize,
    }
    struct TaskInfo {
        id: usize,
        status: TaskStatus,
        call: [SyscallInfo; MAX_SYSCALL_NUM],
        time: usize,
    }
2. 在 trap 和 syscall 模块中添加对应的系统调用编号 const SYSCALL_GET_TASK_INFO: usize = 410; 并且在 config.rs 中添加 MAX_SYSCALL_NUM 配置；然后添加相关的处理过程
3. 在 TASKMANAGER 中添加 get_task_info(id: usize, ts: *mut TaskInfo) 方法，直接对传递进来的原始指针指向的数据进行修改，实际上 TaskInfo 在内核好像没有什么用？？？
4. 在应用程序库中添加 SyscallInfo 以及 TaskInfo 数据结构，然后在 syscall.rs 和 lib.rs 中增加应用程序库 get_task_info 接口

至此大体的框架已经实现，剩下的就是如何统计任务运行总时长，任务使用的系统调用等

统计任务运行总时长：调用 get_time() （这个总时长是指程序创建之后，直到运行结束的全部时长，还是指有效运行时长？？？）
（如果是全部的时长，那么在 TCB 中新增一个任务开始时间 start_time 即可，如果是有效运行时长，那么在产生时钟中断时，即将切换到下一个任务时，在当前任务的 TCB 中time 增加运行时间，除了发生时钟中断，还有任务主动让出 cpu 时也要统计，因此还是需要增加额外信息？？？）
	第二种情况：在 trap 模块对时钟中断进行处理之前，对 TCB 中的 time 进行修改，因此在 TASKMANAGER 中增加一个成员方法来修改任务执行时间，第一种情况处理起来比较麻烦，首先 start_time 分了两种情况，第一种就是运行第一个应用程序，第二种情况就是从第一个应用程序切换到另一个应用程序，这个时候处理 start_time 还需要额外的判断逻辑
    fn record_current_task_time(&self) {
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].time += 10;
    }
任务使用的系统调用次数：系统调用编号在 trap 时可以根据 cx.x[17] 得到，然后可以通过 TASKMANAGER 中增加一个成员方法来修改任务使用的系统调用情况，任务使用的系统调用使用情况，使用 HashMap 应该会速度快一点，使用数组的话，还需要查找
    fn record_current_task_syscall(&self, syscall_id: usize) {
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        let mut idx = 0;
        for i in 0..MAX_SYSCALL_NUM {
            if inner.tasks[current].call[i].id == 0 {
                idx = i;
                break;
            } else if inner.tasks[current].call[i].id == syscall_id {
                idx = i;
                break;
            }
        };
        inner.tasks[current].call[idx].id = syscall_id;
        inner.tasks[current].call[idx].times += 1;
    }
实验结果：
	id:3  status:Ready  time:0			// 此处运行时间还存在问题，因为主动让出 cpu 时没有更新时间信息
	syscall:169 has been used 2 times	// 169 表示 get_time() 系统调用，确实已经运行两次
	syscall:124 has been used 1 times	// 124 表示 yield_() 系统调用，确实已经运行一次
	修改后的结果：
	id:3  status:Ready  time:13777		// 如果采用毫秒为单位，那么会因为运行时间太短，导致 time 为 0
    syscall:169 has been used 2 times
    syscall:124 has been used 1 times
```

### 
