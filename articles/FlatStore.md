
### FlatStore: An Efficient Log-Structured Key-Value Storage Engine for Persistent Memory

#### Abstract

- PM 和高速网卡有希望能够构建高效的 KV 存储
- 小规模访问模式无法匹配 PM 上的持久性粒度，无法完全利用 PM 的带宽
- FlatStore 将 KV 存储进行解耦
	- 持久性的日志存储结构，方便高效存储
	- 变化的索引结构方便快速索引（hash table 核 Masstree 均有实现）
- FlatStore 采用了两种技术，达到低延时、高吞吐量的性能
	- 压缩日志格式，最大化日志中的批处理机会。只在 log 中存储 index metadata 和小的 KV，大的 KV 与 NVM 分配器单独存储，这些大的 KV 不能受到批处理的好处。使用日志操作技术格式化 log 条目，进描述每个操作，而不是记录每个索引的更新，从而达到最小化 index metadata 消耗的空间。不需要在 NVM 分配器中持久化分配元数据，因为在 log 中的 index metadata 和 NVM 分配器中的 metadata 是冗余的。（KV 索引映射了KV 在 NVM 上的地址；而 NVM 分配器记录了所有已经分配的地址）。使用 lazy-persist 分配器管理 NVM 上的空间，在重启后，扫描 log 即可重建 NVM 分配器中的元数据。这种方式使得 log 条目只有 16字节，使得一次能够刷新多个 log 条目。
	- Pipelined Horizontal Batching，在创建批处理时能够从其他的核上窃取日志条目，从而减少了收集日志条目的时间。为了减少核之间的竞争，它精心调度，然核提前释放锁，同时不影响正确性；并且将核编组，用来平衡竞争的开销以及批处理的机会。

#### Introduction

快速数据存储与处理的需求带来了新的问题——如何设计高效的 KV 存储来处理写敏感和小规模的工作负载？

之前的研究表明 KV 存储会产生大量的小规模的写操作，并且其内部的索引结构会造成严重的写放大。
- 基于 hash 的索引，如果 key 产生了冲突或者 hash table 大小变化，则会有大量的条目需要重新 hash
- 基于 tree 的索引同样需要频繁的移动树中的索引条目保持有序，或者进行合并分裂树节点来保证平衡

这些更新只涉及一个或者少量指针的更新。为了保证故障原子性，需要使用额外的刷新指令显示的保留更新的数据。CPU 中的数据缓存粒度只有 64 B，但是 PM 中的块粒度很大，比大多数场景下写的大小大，极大的浪费了硬件的带宽。

解决上述问题的经典方法是采用日志结构来管理 KV 存储，所有的更新可以简单的追加到 log 的尾部。并且，将来自客户端的请求批处理之后再一起进行更新，写的开销就被分摊给各个请求了，但这样的好处受到可以一起执行更新的请求的数量。

但这在基于 NVM 的系统上存在以下问题：
- 只要 IO 的大小比最小的 IO 单元大且有足够的线程能够并发、顺序、随机的写，那么可以达到 Optane DCPMM 上有类似的带宽，但大于单个 IO 单元大小的操作对于批处理不友好。PM 具有更精细（小）的访问力度，如果只在日志中记录每个内存更新，则只能容纳非常有限的日志条目。
- 批处理不可避免的增加延迟，因此在 PM 和 高速网卡中不能采用批处理
因此，一些研究采用了日志结构来保证故障原子性，但是没有采用批处理来分摊开销。

#### Background and Motivation

##### Production KV Store Workloads
- Facebook Memcached pool
- Popular in-memory computation task
- shifting from read-dominated to write-intensive

##### Small Updates Mismatch with Persistence Granularity

经典 KV 存储主要由两部分构成：
- 持久化的索引（tree、hash table）查找 key-value 记录
- 存储管理器（NVM 分配器）存储实际的 KV 对

put 操作涉及的几个步骤
1. 更新实际的 KV（实际情况下大部分的 KV 都是小规模的）
2. 更新分配器内的 metadata
3. 更新索引（造成巨大的写放大）

现代处理器和编译器的优化可能会将存储操作从 CPU 缓存直接刷新到 NVM 中，为了保证故障原子性，KV 存储必须要先预定（order）或者记录更新。但是，这些缓存的粒度远远小于 PM 的粒度。

Figure 1 a：在 Optane 上的 FAST&FAIR 8 字节写操作与原始的 Optane 64字节随机写操作的比较


##### Empirical Study on Optane DCPMM

interesting findings（每个写操作后都紧跟着是 clwb 和 mfence 指令，确保数据写到 Optane DCPMM 内部）
- Figure 1 b：高并发时，顺序和随机访问的带宽相似。从硬件的角度来看，高并发的顺序访问实际上等同于随机访问，每个线程都是在访问不同的地址（图中的带宽比实际带宽低是因为加入了刷新指令）
- Figure 1 c：当刷新同一个缓存行时，将会阻塞 800 ns，可能的两个原因：1）刷新指令实际上是异步的；2）Optane DCPMM 存在片上磨损均衡功能

##### Challenges
- Log-structured storage has less batching opportunities in PM，经过之前的分析，高并发时顺序和随机访问性能接近，因此在 PM 上使用 log 来得到顺序访问的优势并不存在。并且 PM 上的访存粒度比 HDD 和 SDD 上更精细，因此限制了一起刷新时的更新数目。并且日志结构增加了每次更新需要的大小，需要封装额外的 metadata
- Batching increases latency，一方面收集请求需要时间，另一方面批处理与高速网卡的设计原则相悖


#### FlatStore Design
##### Overview of FlatStore（minimal write overhead，low latency，multi-core scalability）
- Compacted OpLog：每个核上一个 OpLog，通过批处理来收集频繁、小规模的更新。为了增加批处理的机会，将每个 log 条目压缩，只存储元数据和小的 KV 对，大的 KV 使用分配器单独存储
- Lazy-persist Allocator：存储大的 KV 条目，因为 OpLog 已经记录了大 KV 的地址（在元数据里面），因此不需要刷新分配器元数据
- Pipelined Horizontal Batching
在 DRAM 中维护动态变化的索引结构

##### Compacted OpLog and Lazy-Persist Allocator

log 条目结构
- 只存储 key 和 ptr
- 存储 key 和 value
更新时，在 log 后追加 log 条目，再更新对应的变化的索引结构；查询时，通过给定的 key 在索引结构中找到对应的条目，避免查找整个 log

log 条目压缩

padding：在批处理之后添加 padding 保证能够 cacheline-aligned

lazy-persist Allocator：使用分配器存储大的 KV 会造成额外的刷新开销，log 条目中的 ptr 记录了已经分配的地址信息，因此可以懒惰的对分配器的元数据进行持久化。而分配器要求支持变长分配，因此采用了 Hoard-like lazy-persist allocator。
- 将 NVM 分配成 4MB 大小的 chunk，再将 4MB 分成不同类别的 data block，同一个 chunk 中的 data block 属于同一个类别，这些数据块的大小信息被记录在每个 chunk 的头部中，用于分配，并且还用 bitmap 来跟踪未使用的 data block。这种设计，使得每个 chunk 是 4MB 对齐且 chunk 头部记录了存储的粒度，因此可以通过 ptr 直接计算出已经分配的 block 在 NVM 中的位置，从而可以恢复 bitmap。
- 考虑到可扩展性，chunk 被划分给不同的核

Put 操作
1. lazy-persist allocator 分配一个 data block，大规模数据进行持久化，小规模跳过
2. 初始化 log 条目并持久化，添加到 log 尾部
3. 在动态变化的索引结构中更新 log 条目相关的入口

冲突一致性：通常需要进行异地更新，因此，除了上述的3个步骤之外，还需要释放原来的块，被释放的块可以立即使用（因为 read-after-delete 异常在 FlatStore 中不会发生）

通过将小型的 KV 对存储在 log 条目中，刷新的次数减少了很多。

##### Horizontal Batching

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


#### Implementation

##### FlatStore-H: FlatStore with Hash Table
索引可以使用任何现有的数据结构。
CCEH：使用单独的 keyhash 划分为多个范围，每个核上有一个实例。
实验结果表明：FlatStore 善于处理扭曲的工作负载。

##### FlatStore-M: FlatStore with Masstree
全局的 Masstree。但 client 仍然可以根据 key 的哈希值选择具体的核。

##### RDMA-based FlatRPC

RDMA：
- send/recv
- read/write

使用 RDMA write 动词在客户端和服务器之间发送请求和响应


#### Evaluation

图7
- 小规模的工作负载下， FlatStore 有更高的吞吐量。主要原因：1）其他两个每次更新都需要刷新索引条目、bitmap、和分配器中的 KV 记录；2）当两个 key 冲突时，Level-Hashing 需要 rehash 相关的条目，而 CCEH 需要分裂，会放大刷新时间；3）Pipelined HB 合并小规模的日志条目，通过批处理持久化，减少了写的次数
- 天然适合扭曲的工作负载，不忙的核可以成为 leader 将忙碌的核中的请求收集进行批处理，而大规模的 KV 工作负载不再具有优势，因为它们仍然是在各自的核上进行持久化。

图8
- FlatStore-M 比 FlatStore-H 更好，加速比更高，tree 需要进行分裂合并，会增加写放大。
- FlatStore-M 比 FlatStore-FF 好，FlatStore-FF 比其他的好


##### Multicore Scalability
