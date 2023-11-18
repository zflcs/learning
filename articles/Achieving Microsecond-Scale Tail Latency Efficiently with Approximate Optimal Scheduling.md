#### Achieving Microsecond-Scale Tail Latency Efficiently with Approximate Optimal Scheduling

对现有的调度策略进行近似，打破了在长尾延迟和吞吐量之间的权衡。


##### 系统模型

1 调度线程 + n 工作线程

三个主要影响吞吐量开销的因素

1. 抢占调度
2. 调度线程与工作线程之间单队列同步通信开销
3. 调度线程开销（调度线程不运行任何 app 逻辑）


##### 设计

关键技术

1. 编译器强制调度线程与工作线程之间协作异步通信（异步导致抢占不精确，但不会对尾部延迟产生显著影响，且使得抢占开销显著减小）
2. JBSQ 调度将局部队列模拟成全局队列，降低缓存开销
3. 调度线程在所有工作线程处于忙碌状态时，从全局队列中窃取任务

###### 编译器强制协作

通过 cache 行进行通信，不通过 IPI。调度线程在 cache 行中写入抢占标志，而工作线程定期检查 cache 行中的抢占标志，一旦收到信号即向 cache 行中写标志通知工作线程已经进行抢占。
由于调度线程可以看到系统中所有的请求，因此可以较方便的实现其他的调度策略。
通过 cache 行通信，可以将开销降低至 2 个 cycles。cache 行 miss 会导致 150 个 cycles 的开销。

Safety-first preemption：不会出现工作线程持有锁时发生抢占的情况。Shinjuku 在 API 调用时禁用了抢占。（值得借鉴）

###### Stall-Free Workers

避免工作线程空闲，使用 JBSQ（K） 模拟单队列，（K 表示工作线程局部队列中消息上线）但比使用 SQ 的开销低 9-13x（JBSQ 掩盖了线程间通信的缓存一致性开销）

###### Work-Conserving Dispatcher

在所有工作线程处于忙碌状态时，调度线程参与消息处理。（通过 rdtsc() 实现自我抢占，定期检查探针周期性检查是否需要从处理 app 请求切换去进行调度，一旦调度线程处理请求，则会持续处理直到完毕，中间如果需要抢占，则把未处理完的请求保存在专用缓冲区，下次继续处理）

##### Concord 原型

###### API

```c
setup()
setup_worker(int core_num)
response_t* handle_request(request_t*)
```

##### 评估

1. 在资源有限的情况下，让调度线程参与处理请求有效
2. 在低负载的情况下，尾部延迟略微增加（调度线程窃取请求）
3. 需要获取源码
4. 单个调度线程，在负载更高的场景下需要扩展成多个调度线程


