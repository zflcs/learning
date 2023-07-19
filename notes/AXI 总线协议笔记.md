
#### 五个通道

写请求通道：aw_xxx
写数据通道: w_xxx
写响应通道: b_xxx
read request channel: ar_xxx
read data channel: r_xxx

单独使用写响应信号完成对写事务的响应，经过缓冲，不需要等待上一个写事务的响应才能进行下一个写事务。读事务的响应与所读数据一起返回。


