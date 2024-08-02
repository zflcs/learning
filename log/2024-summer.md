# 2024 暑假日常

## TODO

- [x] 完成对 starry 的修改
- [x] 处理 ZJP 对 rCore-N 的修改
- [ ] 前两篇论文工作的总结，以及后续投稿

## 20240802

- [x] 修改共享调度器小论文

## 20240801

- [x] 修改共享调度器小论文

## 20240731

- [x] 阅读论文：Revisiting Coroutines
- [x] 修改小论文

## 20240730

- [x] 阅读论文：Light-weight Contexts: An OS Abstraction for Safety and Performance
- [x] 阅读论文：Revisiting Coroutines

## 20240729

- [x] debug
  - 栈的保存和使用还存在问题
  - 在回到 yield 函数后，将栈清空，这是正确的顺序，而不是在恢复之前将其清空
  - 运行 async_monolithic 时，在 cfs 和 rr 调度器下，都能够正常工作，反而在 fifo 调度器下不能正常工作，问题是 main_coroutine 被莫名其妙的 block 了，还是因为没有正确的回收栈的问题
  - 栈池的管理需要一个合理的方式
  - 指针乱飞的问题仍然没有解决，在不乱飞的情况下，已经可以执行 ls 指令了
- [x] 把 overleaf 上的论文拉到 github 仓库中，方便进行 latex-diff，并对格式进行了整理
- [x] 整理 rCore-N 仓库
  - 已进行合并，但代码运行还存在问题，出现 bug，将其放置到新的分支 fixed-warnings 上
- [x] 修改共享调度器小论文
  - 找到两篇与调度相关的论文：
    - Efficient Scheduler Live Update for Linux Kernel with Modularization
    - Light-weight Contexts: An OS Abstraction for Safety and Performance

## 20240728

- [x] debug
  - 在上下文嵌套时，需要将 prev_ctx_ptr 字段设置为 0，但还是出现了指针乱飞的情况
  - load_next_ctx 恢复上下文时，存在问题
  - 针对在 async 环境下，使用同步的 yield 接口进行单独测试
  - yield 接口存在问题，save 或者 load 时存在问题，导致 pc 跑飞，在 fifo 调度器（没有抢占）的情况下，跑 1000万次测试正常
  - save 保存了上下文之后需要跳转到 schedule_with_sp_change 函数，但直接跳转执行了这个函数的中间部分的代码，没有从入口开始执行
  - 使用 fifo 调度器正确跳转了，但使用 cfs 调度器则跳转至错误的地方
  - 在 yield 调用 save 之前需要设置当前的任务不能被抢占，否则会有一定概率出错
  - 在发生上下文嵌套时，等下一次恢复时，嵌套的指针不正确

## 20240727

- [x] debug
  - 使用线程，前后两次 yield 的 ra 不一致，正常现象，在中间调用了其他函数，改变了 ra，yield 上下文切换的前后，sp 没有变化，证明逻辑是正常的
  - 确定了是指针乱飞，莫名其妙的将 main_coroutine 阻塞了，在运行 idle 后，尝试将 main_coroutine 占用的资源释放掉
  - 在 yield 之前增加了打印语句后，无法出现 bug，导致调试异常困难
  - 在 yield 前后打印上下文指针中的内容，发现并没有按照想象的代码解决上下文嵌套的问题，因此，在上下文嵌套这里需要仔细检查
- [x] 参加 vivo 蓝河 os 技术沙龙

## 20240726

- [x] debug，找出切换过程中存在的问题
  - 在多核 fifo 调度器下，出现了其中一个任务试图释放另一个任务正在占用的锁，但这两个任务是在同一个核上运行，主要的问题还是出在切换的逻辑上
  - 在加入了 Utrap 上下文类型切换后，通过系统调用进入到内核后，会重新打开中断，导致会发生时钟中断，这个过程可能会被再次打断，这种情况下必须要考虑上下文嵌套的影响，会导致指针乱飞，不能恢复正常的执行
  - 出现了 PC 指针乱飞的情况，main_coroutine 不应该会出现调用同步的 wait 接口，但莫名其妙的调用了 wait，将自己阻塞了，并且其他核也莫名奇妙的调用了 wakeup_task 函数，应该还是线程切换时，上下文没有正确保存
