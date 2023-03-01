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




#### How is the system designed?

##### Compact log
log 只用来存储索引的元数据和小规模的 KV 对，使用 operation log 技术格式化 log 条目，而不是记录每个索引的更新。
大规模 KV 使用 NVM 分配器单独存储，并且不需要持久化 NVM 分配器的元数据。

log 条目结构
- 只存储 key 和 ptr
- 存储 key 和 value
更新时，在 log 后追加 log 条目，再更新对应的变化的索引结构；查询时，通过给定的 key 在索引结构中找到对应的条目，避免查找整个 log


padding：在批处理之后添加 padding 保证能够 cacheline-aligned

##### lazy-persist allocator
使用分配器存储大的 KV 会造成额外的刷新开销，log 条目中的 ptr 记录了已经分配的地址信息，因此可以懒惰的对分配器的元数据进行持久化。而分配器要求支持变长分配，因此采用了 Hoard-like lazy-persist allocator。
- 将 NVM 分配成 4MB 大小的 chunk，再将 4MB 分成不同类别的 data block，同一个 chunk 中的 data block 属于同一个类别，这些数据块的大小信息被记录在每个 chunk 的头部中，用于分配，并且还用 bitmap 来跟踪未使用的 data block。这种设计，使得每个 chunk 是 4MB 对齐且 chunk 头部记录了存储的粒度，因此可以通过 ptr 直接计算出已经分配的 block 在 NVM 中的位置，从而可以恢复 bitmap。
- 考虑到可扩展性，chunk 被划分给不同的核

Put 操作
1. lazy-persist allocator 分配一个 data block，大规模数据进行持久化，小规模跳过
2. 初始化 log 条目并持久化，添加到 log 尾部
3. 在动态变化的索引结构中更新 log 条目相关的入口

冲突一致性：通常需要进行异地更新，因此，除了上述的3个步骤之外，还需要释放原来的块，被释放的块可以立即使用（因为 read-after-delete 异常在 FlatStore 中不会发生）

通过将小型的 KV 对存储在 log 条目中，刷新的次数减少了很多。

##### pipelined horizontal batching
- 减少竞争，保证正确性的前提下，提前释放锁
- 对 server core 进行编组，使得负载均衡
vertical batching：batches 只收集所在的核上的请求，尽管减少了持久化开销，但是高延迟的负面影响。会减少每个核上进行批处理的机会，

Pipelined Horizontal Batching (Pipelined HB)：在 Naive HB 的基础上发展而来。采用了以下方式：
- 核之间的全局锁保证同步
- 每个核的请求池可和之间通信
分三个阶段进行持久化：
- l-persist：分配空间和持久化 KV 条目
- g-persist：持久化 log 条目
- volatile phase：更新变化的索引结构
一旦某个核没能获取到全局的锁，就会轮询下一个到达的请求，开始执行第二个 Horizontal Batching。而获取到锁的核称为 leader core，它从其他的核收集了 log 条目之后立即释放锁。没有获取到锁的核异步的等待上一个 lead 核 完成的消息。
Discussion：会对请求重新排序，因此在之后的 get 操作访问之前的 put 操作相同的 key 的数据可能会失败。为了解决这种问题，每个核会维护一个独占拥有的冲突队列来跟踪正在处理的请求。任何冲突的 key 的请求会被延迟。

Pipelined HB with Grouping：大量的核竞争全局锁时，会导致大量的同步开销。对几个核进行编组，组内的核竞争一个锁，几个组之间竞争全局锁。将同一个 socket 的核编组效果最佳。

##### Log Cleaning
避免 OpLog 随意增长，需要进行压缩，丢弃掉过时的 log 条目。每个核维护一个内存中的 table 来跟踪 OpLog 中指向的 4MB 的 chunk。
每个组运行一个后台线程清理日志，因此，组与组之间可以并行。chunk 被回收取决于其内部存活的条目的比例和空闲块数量。在回收时，检查 reclaim list 找到 victim chunks。只需要比较 log 条目中的 version 字段与内存中索引即可判断。一个 chunk 内活跃的 log 条目会被复制到新分配的 NVM chunk 中。


##### Recovery
Recovery after a normal shutdown
在关机之前，将变化的索引结构复制到 NVM 预先规定的区域，之后刷新每个 chunk 的 bitmap。再写入一个 shutdown 标记表示是正常关机。

Recovery after a system failure
shutdown 标记不合法，则需要扫描 OpLog 恢复索引结构和 bitmap。1 billion（十亿） KV 条目只需要 40s。
还支持 checkpoint

### Results

##### Empirical Study on Optane DCPMM
结论：（每个写操作后都紧跟着是 clwb 和 mfence 指令，确保数据写到 Optane DCPMM 内部）
- 高并发时，顺序和随机访问的带宽相似。从硬件的角度来看，高并发的顺序访问实际上等同于随机访问，每个线程都是在访问不同的地址（图中的带宽比实际带宽低是因为加入了刷新指令）
- 当刷新同一个缓存行时，将会阻塞 800 ns，可能的两个原因：1）刷新指令实际上是异步的；2）Optane DCPMM 存在片上磨损均衡功能

#### How is the design evaluted, what are the key results?


- 小规模的工作负载下， FlatStore 有更高的吞吐量。主要原因：1）其他两个每次更新都需要刷新索引条目、bitmap、和分配器中的 KV 记录；2）当两个 key 冲突时，Level-Hashing 需要 rehash 相关的条目，而 CCEH 需要分裂，会放大刷新时间；3）Pipelined HB 合并小规模的日志条目，通过批处理持久化，减少了写的次数
- 天然适合扭曲的工作负载，不忙的核可以成为 leader 将忙碌的核中的请求收集进行批处理，而大规模的 KV 工作负载不再具有优势，因为它们仍然是在各自的核上进行持久化。


- FlatStore-M 比 FlatStore-H 更好，加速比更高，tree 需要进行分裂合并，会增加写放大。
- FlatStore-M 比 FlatStore-FF 好，FlatStore-FF 比其他的好

##### Multicore Scalability


#### What question do you have?









