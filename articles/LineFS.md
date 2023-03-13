### problem

#### What problem is the paper solving?
分布式文件系统（DFS）与 App 争用 CPU，导致两者的性能下降。而将 DFS 的功能卸载到 SmartNIC 来降低 CPU 开销是当下的主流趋势，但是 SmartNIC 的处理性能较难满足 DFS 的需求。

#### Why is this problem important?
1. 引入了 PM 之后，主机 CPU 的开销延迟不能忽略，逐渐成为瓶颈。
2. SmartNIC 的功率只有 25W，只有低频的 core 和小规模的内存缓存，并且通过 PCIe 访问 PM 的延迟在微秒级别，主机 CPU 通过 DDR 访问 PM 的延迟在 100ns 级别，因此简单卸载到 SmartNIC 反而会导致延迟增加

### Context

#### What was the previous state of the art?

新的 DFS 设计针对了 PM 进行优化，使用 client-local PM，但相比于 client-server 的设计，这种方式要求本地的文件系统管理任务，与 app 竞争资源，耗费大量的 CPU 资源。

#### How does the paper advanced the state of the art?

LineFS 将 CPU 密集型的任务（备份、压缩、数据发布、索引和一致性管理）卸载到 SmartNIC 上。并对发布和备份操作进行并行、批处理。异步的优化，降低了延迟。

### Design

#### What are the key insight from the design?

构建了 persist-and-publish 并行流水线模型，将 PM 存储操作从关键路径上隔离开来，使其能够异步执行。降低由卸载到 SmartNIC 上带来的延迟。

#### How is the system designed?

LineFS 谨慎的将 DFS 组件分布在主机和 SmartNIC 上，LibFS 使用快速的主机 core 在主机 PM log 上进行持久化，kernel worker 将私有 log 发布到公共的 PM 上；而 NICFS 在后端使用 SmartNIC core 进行发布和备份。

NICFS 上则建立 persist-and-publish 并行流水线模型：主机上的 LibFS 将增长至 chunk 大小的 log 通过 RPC 发送给 NICFS，而 NICFS 将 publish 和 replication 划分成几个执行阶段（增加压缩数据的阶段，减少占用的网络带宽），并行执行高延迟的操作，并且通过批处理来分摊 PCIe 的高延迟开销。通过这种方式让 app 能够继续运行，减少 DFS 与 app 之间的资源竞争。

LineFS 还针对 RDMA 进行优化，针对低延迟和高吞吐量的的连接进行不同的处理，还有多路复用RDMA降低QPs。

### Results

#### How is the design evaluted, what are the key results?

写操作
replicas idle：
在 replicas idle 的情况下，Assies 性能最差，而 Assise-BgRepl 则有 124% 的提升，而 LineFS 比 Assies 高 133%，比 Assies-BgRepl 高 4%。要达到相同的带宽，Assies 需要更过的 client。
LineFS-NotParallel 与 LineFS 比较，其性能至少下降了 60%
replicas busy：
在 replicas busy 的情况下，当只有一个 client 时，所有的实现与空闲的情况下相似，随着 client 增加，DFS 会使用更多的主机 CPU 资源和内存资源，LineFS 的资源竞争最小，比其他 DFS 性能高出 33%。但是 LineFS 使用了 kernel worker 进行发布操作，会导致性能下降。

读操作，因为 LineFS 并没有卸载到 SmartNIC，所以它和 Assies 性能相似。

通过运行 IO 和 CPU 密集型的工作负载比较两者主机上的 streamcluster 受到的性能干扰，最终 LineFS 有最高的吞吐量，并且 streamcluster 在主机和副本上运行的性能下降了 49% 和 19%。

kernel worker 是 LineFS 影响 app 性能的主要原因，使用最高比重的主机发布方法的 LineFS 对 streamcluster 的性能影响最大。使用 No copy 的策略不会对 streamcluster 产生影响，使用 DMA 中断 + 批处理的方式相比于 No copy 下降了 23%，而 CPU memcpy 会使其下降 61.5%。

在 Tencent Sort 测例上测试 LineFS 压缩数据对网络带宽的影响，当压缩比例为 80% 时，LineFS 性能比 Assies 高 10%，但能够节省 72% 的网络带宽。

#### What related problems are still open?

是否可以将 kernel worker 理解为 LineFS 故障处理的关键，如果是因为 kernel worker 本身的故障，那么系统应该如何保证可用性。

