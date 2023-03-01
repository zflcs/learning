### problem

#### What problem is the paper solving?
很少有研究将 flash 的内部并行性运用在文件系统层面。即使是针对 flash 优化过的文件系统，也存在严重的垃圾回收问题。

文章的主要贡献是在文件系统层面利用了 flash 的内部并行性，提高了垃圾回收的效率。同时，设计了并行感知调度机制，增加了性能的稳定性。

#### Why is this problem important?
Flash 内部的优化对于上层的文件系统是透明的。会导致以下几种问题：
- 在 FS 层面和在 flash 内部的 FTL 层面的优化可能会互相冲突。
- 在 FS 层面和在 flash 内部的 FTL 层面的日志结构管理机制会降低垃圾回收的效率。
- FS 层面和在 flash 内部的 FTL 层面，相互隔离的 I/O 请求调度会导致不稳定的 I/O 延迟


### Context

#### What was the previous state of the art?
以往的研究从以下几个方面针对 flash 进行优化：
- 移除掉 FS 层和 FTL 层面冗余的功能，改进数据分配性能。
- 使用基于对象的 FTL 层，缩小了 FS 和 FTL 层的语义差别，帮助软硬件协同设计
- 利用 flash 存储器的特点，重新设计高效的元数据机制
- 研究 flash 读写不平衡的特点，设计高效持久化的文件系统目录树
- 针对 flash 优化的文件系统 F2FS 合并到 Linux 内核中
但是仍然存在着以下问题：
- 文件系统为了友好支持flash设备，通常会进行冷热数据分离，即把数据分为hot group和cold group。与此同时，flash设备为了考虑带宽，将会最大化请求数据的并行性。于是FS层分好的data group会在存储设备端倍分到不同的parallel units中。
- flash-friendly FS，如F2FS，垃圾回收的单位是segment，而segment对应的flash page可能会分布在多个flash的并行单元中。这将会降低FTL中GC的效率。且FS层invalid的block对于FTL层而言无法感知，这回导致FS和FTL中GC的部分冲突。
- 由于flash内部erase的操纵延迟比read/write高一个数量级，所以部分erase操作可能会导致后续的read、write请求的延迟剧烈波形，这对延迟敏感型的应用而言是致命的缺点。而这本可以通过合理的并行单元调度来解决。

#### How does the paper advanced the state of the art?
ParaFS 相比于已有的研究，其进步主要体现在针对内部并行性的运用，同时保证了高效的垃圾回收以及稳定的性能。
- 向 FS 暴露出简化的块级 FTL 信息
- FS 层和 FTL 层的垃圾回收机制相互协作
- 针对请求进行调度，保证稳定的性能

### Design

#### What are the key insight from the design?
最核心的设计是将 flash 内部的 FTL 进行简化，使得上层的文件系统能够感知到 flash 的信息。
- 通过三个参数向文件系统暴露出设备信息：flash 通道数量、flash 页面大小、flash 块大小
- 使用静态的块级映射表，只有在进行磨损均衡或者某个块故障时，才会对这个表进行重新映射
- FTL 进行垃圾回收时，只需要对块进行擦除，不需要对合法的块进行复制

#### How is the system designed?
ParaFS的设计主要使用了以下三种技术。一是 2 维数据分配，二是协同垃圾回收，三是并性感知调度机制。

##### 2 维数据分配
在 FS 层面，每个 region 对应着 flash 中的通道，region 内部再划分为若干个 segments（对应着 flash 内部的 block），而 segments 内部划分为若干个 data page（对应 flash 内部 page）。同时，每个 region 内部存在多个分配器，用于分配 segment。
为了解决文件系统内的冷热数据划分与 flash 通道级别的并行性之间的冲突，ParaFS 从两个维度进行数据分配。
- 通道并行维度：将请求的数据暂存到 DRAM 中，以 data page 的粒度将其划分到不同的 region 中，保证并行性
- 数据冷热维度：根据划分的 data page 数据的冷热程度，将数据分配到冷热程度相似的 segment 中，保证同一个 segment 中的数据冷热程度相似。

##### 协同垃圾回收机制
ParaFS 中的垃圾回收由 FS 和 FTL 层的垃圾回收协同完成。当触发垃圾回收机制时，FS 层的垃圾回收启动一个 paraGC 线程，将 victim segment 中有效的 data page 移动到其他的空闲空间中，之后告知 FTL 中的垃圾回收机制，直接擦除对应的 block

##### 并行感知调度机制
ParaFS 在 FS 层面对 read/write/erase I/O 请求进行调度，每个 region 配置了一个调度器。依据请求的类型，在 dispatching 阶段，将写请求分配到压力最小的调度器上；在 scheduling 阶段，根据设置的 read 和 write/erase 的比例分配时间片，获取稳定的性能。

### Results

#### How is the design evaluted, what are the key results?

##### 写流量轻的情况
- ParaFS 在所有场景下性能比其他 FS 好。
- BtrFS 性能差是因为频繁的同步操作会造成 wandering tree 问题
- 其他 FS 性能类似，性能瓶颈主要是 PCIe 驱动，写流量较轻时，垃圾回收机制的影响较小。
- 通道数量越多，ParaFS 性能越好

##### 写流量重的情况
- 因为 F2FS 的优化会与 flash 内部并行性冲突的原因，Ext4 性能比 F2FS 好
- 因为垃圾回收负担的增加，F2FS_SB 比 F2FS 性能好
- ParaFS 性能比其他的 FS 更好

##### 写流量与垃圾回收
- ParaFS 的垃圾回收数量更少且更高效
- ParaFS 在 FS 层面写的流量会更多，但在 FTL 层面写的流量更少

##### 性能稳定性
- 并行感知调度机制保证性能更加稳定


#### What question do you have?
发生写请求时，数据先缓存在内存中，之后再根据适当的实际刷新到 flash 中，这时进行二维数据分配，判断 hotness 相似的标准是什么？当同时出现大量写请求时，是否会出现实际的 hotness 与分配的 segment hotness 不一致的情况？