### problem

#### What problem is the paper solving?

DFS 与 App 争用 CPU，导致 App 的性能下降。
DFS 将部分工作卸载给 SmartNIC 来处理是解决 CPU 开销增大的一个策略，但是 SmartNIC 的处理性能较难满足 DFS 的需求。



#### Why is this problem important?
1. 增加了访问主机内存的延迟
2. CPU 通过 DDR 访问 PM 的延迟在 100ns 级别，而通过 PCIe 访问 PM 的延迟在微秒级别
3. SmartNIC 的功率只有 25W，只有低频的 core 和小规模的内存缓存


### Context

#### What was the previous state of the art?
但是，现有的解决方案都没有考虑完整 DFS 的卸载需求。
新的 DFS 设计针对 PM 进行优化，使用 client-local PM，但相比于 client-server 设计，这种方式要求本地的文件系统管理任务，与 app 竞争资源。

#### How does the paper advanced the state of the art?
将 DFS 的操作分解成各个执行阶段。而这些阶段可以被卸载到 SmartNIC 上的并行数据路径执行管道。

LineFS 将 CPU 密集型的任务（备份、压缩、数据发布、索引和一致性管理）卸载到 SmartNIC 上。

支持 client-local PM。

### Design

#### What are the key insight from the design?

构建了 persist-and-publish 模型，将 PM 存储操作从关键路径上隔离开来，使其能够异步执行。
引入并行数据通路执行流水线，保证提供强的文件系统一致性的同时加速异步发布操作。

#### How is the system designed?
- 采用 client-local DFS 模型，在客户端机器上执行部分 DFS 功能，避免 client-server 访问 PM 的通信延迟。
- 谨慎的将 DFS 组件分布在 host 和 SmartNIC 上。

LineFS 节点由两个组件构成：LibFS（LineFS client running on the host）、NICFS

利用并行、批处理和异步操作隐藏 SmartNIC 执行和数据访问的开销。

persist-and-publish：

pipeline parallelism：




### Results

#### How is the design evaluted, what are the key results?




#### What question do you have?
