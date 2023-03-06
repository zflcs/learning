### problem

#### What problem is the paper solving?
快速数据存储与处理的需求带来了新的问题——如何设计高效的 KV 存储来处理写敏感和小规模的工作负载？KV 存储中的小规模的访问模式与 PM 设备上持久化的粒度不匹配，导致没能很好的利用 PM 设备的带宽。

#### Why is this problem important?
- KV 存储中的索引结构会导致写放大（大多数更新操作只涉及少数指针）
	- hash 索引结构需要重新 hash
	- tree 索引结构涉及节点合并分裂，移动节点保持有序
- 为了保证故障原子性，需要使用额外的刷新指令使数据持久化，但是 cache 的粒度只有 64B，PM 设备的粒度是 256B，远远比写的数据大

### Context

#### What was the previous state of the art?
解决这种粒度不对称的经典方法就是采用 log 进行管理，并且利用 batch 对大量的请求同时进行更新。
但这种方式在 NVM 上不适用：
- 只要 I/O 大小比 NVM 上的最小 I/O 单元大，且有足够的线程并发执行 I/O，在 NVM 上顺序和随机读写的带宽相似，因此不适合用 batch 比单个I/O 单元更多的数据。（若 batch 的处理数据由很多个随机的最小 I/O 单元组成，那么这种顺序处理随机写性能会下降）
- NVM 上的最小 I/O 单元的粒度只能容纳非常有限的 log 条目（记录内存更新）
- batch 会增加延迟，与高速网卡和 PM 的设计相违背

因此对于这方面已有的研究只采用了 log 结构保证故障时的原子性，减小内存碎片化，并没有利用 batch 分摊持久化的更新。

针对索引结构：
- 整个索引存放在 NVM 上
- 在 NVM 上存储树的叶子节点，内部节点存储在内存中
- 在内存和 NVM 上各自维护索引结构，后台线程进行同步

#### How does the paper advanced the state of the art?
一方面是将索引与持久化进行解耦
一方面是利用 log 和 batch 进行高效的持久化
- 构造了 compact log 格式，使得 NVM 的最小 I/O 单元可以容纳更多的 log 条目
- 利用 pipelined horizontal batching 技术解决 batch 延迟的问题

### Design

#### What are the key insight from the design?
核心的设计是 pipelined horizontal batching，因为即使对 log 进行了压缩，刷新时的粒度仍然是 cache 的粒度，仍然不能利用 NVM 的带宽。

#### How is the system designed?

##### Compact log
log 只用来存储索引的元数据和小规模的 KV 对，使用 operation log 技术格式化 log 条目，而不是记录每个索引的更新。大规模 KV 使用 NVM 分配器单独存储，并且不需要持久化 NVM 分配器的元数据。

log 条目结构划分为 ptr-based 和 value-based 两种类型，每个 log 条目占 16 B，因此 cacheline 可以包含 4 个 log 条目，一次 batch 处理可以刷新 16 个 log 条目。同时每个 cacheline 后添加 padding 保证 cacheline 对其，避免两次 batch 处理中存在对同一个 NVM 区域进行更新，减少延迟。

##### lazy-persist allocator
log 条目中的 ptr 记录了已经分配的地址信息，因此可以懒惰的对分配器的元数据进行持久化。并且将 NVM 块分成不同的大小，支持变长分配。

先将 NVM 分配成 4MB 大小的 chunk，再将 4MB 分成不同大小类别的 data block。同一个 chunk 中的 data block 属于同一个类别，这些数据块的大小信息被记录在每个 chunk 的头部中，用于分配，并且还用 bitmap 来跟踪未使用的 data block。这种设计，使得每个 chunk 是 4MB 对齐且 chunk 头部记录了存储的粒度，因此可以通过 ptr 直接计算出已经分配的 block 在 NVM 中的位置，从而可以恢复 bitmap。

##### pipelined horizontal batching
每个 server core 能够从其他 core 窃取 log 条目，所有的 core 竞争一个全局锁保证同步互斥，并且为每个 core 建立一个可以核间通信的请求池

将 put 操作分三个阶段进行持久化：
- l-persist：分配空间和持久化 KV 条目
- g-persist：持久化 log 条目
- volatile phase：更新变化的索引结构

一旦某个核没能获取到全局的锁，就会轮询下一个到达的请求，开始执行第二个 Horizontal Batching。而获取到锁的核称为 leader core，它从其他的核收集了 log 条目之后立即释放锁。没有获取到锁的核异步的等待上一个 lead 核 完成的消息。

当前一个 put 操作还在 leader core 上处理，之后，对于同一个 key 的 get 操作不会分配到其他的 server core，而是在同一个 core 中，用冲突队列进行跟踪，对于冲突的操作请求，将会被推迟。

大量的核竞争全局锁时，会导致大量的同步开销。对几个核进行编组，组内的核竞争一个锁，几个组之间竞争全局锁。

##### Log Cleaning
每个核维护一个内存中的 table 来跟踪 OpLog 中指向的 4MB 的 chunk。当 chunk 内部存活的条目的比例或者空闲块的数量达到一定比例时将会被回收。

每个 HB 组运行一个后台线程清理日志，因此，组与组之间可以并行。在回收时，只需要比较 log 条目中的 version 字段与内存中索引即可判断是否存活。一个 chunk 内活跃的 log 条目会被复制到新分配的 NVM chunk 中，在将内存中的索引指向新的位置，旧的块被回收。chunk 的地址被记录在 journal field。

##### Recovery
Recovery after a normal shutdown：在关机之前，将变化的索引结构复制到 NVM 预先规定的区域，之后刷新每个 chunk 的 bitmap。再写入一个 shutdown 标记表示是正常关机。

Recovery after a system failure：shutdown 标记不合法，则需要扫描 OpLog 恢复索引结构和 bitmap。

### Results

#### How is the design evaluted, what are the key results?
Hash-based、Level-Hashing、CCEH 与 FlatStore-H 比较（每个核一个 Level-Hashing、CCEH；Hash-based 创建足够大的空间，在重新 hash 之前统计性能）；FPTree（STX B+树）、FAST-FAIR 与 FlatStore-M 比较。

YCSB 上的 put 操作性能对比
FlatStore-H：
- FlatStore-H 吞吐量更高：1）其他两个每次更新都需要刷新索引条目、bitmap、和分配器中的 KV 记录；2）当两个 key 冲突时，Level-Hashing 需要 rehash 相关的条目，而 CCEH 需要分裂，会放大刷新时间；3）Pipelined HB 合并小规模的日志条目，通过批处理持久化，减少了写的次数。由于采取的冲突策略不同，CCEH 比 Level-Hashing 稍高
- 在极端的测试场景下，FlatStore-H 性能表现更明显：1）Level-Hashing 和 CCEH 就地更新索引结构；2）pipelined HB 的并行性。在大规模 KV 场景下，pipelined HB 无效了。

FlatStore-M：
- 比 FlatStore-H 性能更明显（与其他两者的对比）
- 由于 Masstree 的设计，FlatStore-M 比 FlatStore-FF 吞吐量更高；FlatStore-FF 比 FPTree 和 FAST-FAIR 高

Facebook ETC Pool
- FlatStore-H 和 FlatStore-M 均比其他的性能更好，FlatStore-M 更甚，因为 Masstree
- 在 read/write 比例增大时，FlatStore-H 的性能与其他两者相似，而 FlatStore-M 性能仍比较突出，这是因为基于树，put 操作的开销不可忽略

Multicore Scalability
普通情况和极端情况下，在 core 达到 26 之前，几乎线性增长；当 core 继续增大时，PM 的带宽成为了瓶颈。

Analysis of Each Optimization
- compacted OpLog：（不使用 pipelined HB，仅实现 log 结构且一次只处理一个请求）平均比 CCEH 高 29.3%；
- pipelined HB：（Naive HB 能窃取其他核的请求，但是不提前释放锁）在 value 较大时，一次刷新的 log 减少了，导致分摊下来的开销增大了
- pipelined HB on latency：（Vertical Batching）client Batchsize 小的，两者性能相似，越大，Vertical Batching 延迟越高，同样延迟的情况下，pipelined HB 的吞吐量高

Garbage Collection Overhead
当触发 GC 时，性能只下降 10%，性能和 log 清理的速度都保持稳定：后端进程处理，不会阻塞处理 client 的请求；OpLog 轻量级，维护索引元数据的开销小；每个组的 GC 可以并行












