# 实验四报告

### 重写 sys_get_time

> 为什么引入了虚拟内存机制，sys_get_time 系统调用就失效了？？？
>
> 好像没有失效？？？

### 创建 mmap 和 munmap 匿名映射

> 首先判断 start 和 port 是否符合条件，然后获取当前任务的地址空间 token，current_user_token（）
>
> 然后根据 token，得到页表
>
> 然后判断 [start, start + len) 之间是否有被映射的区域
>
> 接下来构建一个 map_arae，利用提供的 map 和 unmap 接口

