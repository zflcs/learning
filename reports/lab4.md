# 实验四报告

### 重写 sys_get_time

> 为什么引入了虚拟内存机制，sys_get_time 系统调用就失效了？？？
>
> 好像没有失效？？？

### 创建 mmap 和 munmap 匿名映射

> 首先判断 start 和 prot 是否符合条件，然后在当前任务的地址空间 memory_set 中利用 insert_framed_area 直接插入到 memory_set.area[0] 中
>

