# 实验三报告

### 实践作业：增加系统调用获取任务信息

```
思路：
	声明一个全局性的 TaskInfo 数据结构，交给 TaskManager 来管理，在 TaskManager 中增加相应的成员方法来统计任务使用的系统调用以及调用次数、任务总运行时长等信息
	分析 TaskInfo 这个类的成员方法，需要打印出任务信息，因此要有 print 方法，如果只是想要打印出来，那么直接在内核打印即可，不用返回数据到用户态，然后再调用系统调用来打印，其余的方法应该是在 syscall 模块中来实现
实现：
1. 创建 SyscallInfo 和 TaskInfo 数据结构
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
2. 实现相关的成员方法 new(), print_task_info()
3. 暂定系统调用的编号为 SYSCALL_GET_TASK_INFO = 200
4. 
```

### 问答作业

```

```

