[TOC]

### Rethinking File Mapping for Persistent Memory

#### Abstract

Persistent main memory 大幅提升了 IO 性能，但是实际上 70% 的 IO 通道性能用于处理 file mapping（将文件 offset 映射到物理位置），即使经过 PM 优化过后的文件系统，也是基于几十年前设计的 file mapping。

文章提出了一种基于 hash 的 file mapping 策略。HashFS 使用单一的 hash 操作来完成所有的映射和分配操作，绕过了文件系统缓存，通过 SIMD 并行性预取并且显示缓存转换。

#### Instruction

PM 可以按字节寻址，只比 DRAM 的延迟高 2 ~ 3 倍，对文件系统设计者具有很强的吸引力，但是很少关注 file mapping。现存的 PM 优化文件系统要么简单的使用针对较慢的块设备的 file mapping，或者没有是使用没有严格分析的新方法。

目前尚不清楚在 DRAM 中维护映射结构的易失性 copy 是否比利用 CPU 缓存来缓存映射结构或者设计专门的文件映射缓存更有益。

PM 的字节寻址能力可以设计完全随机访问的映射结构（例如 hash 表）

PM file mapping 的设计与文件系统无关，可以用在其他的 PM 文件系统中

- 在 Strata 设计并实现了4种 file mapping 方式

- HashFS 在 Strata‘s page-cached extent trees YCSB 工作集上增加 45% 的吞吐量

- 传统的 page-cache 对于 PM 优化的 file mapping 没有任何好处，PM 优化过的 storage structures 不适合用作 file mapping 结构

#### File Mapping Background

file mapping 是将一个文件的逻辑偏移映射到底层设备的实际物理位置。file mapping 通过在文件系统中维护一个或多个 metadata 数据结构（file mapping structure）来实现，这些结构在固定的粒度上将逻辑地址（文件 + 偏移）映射成物理地址（设备偏移）。

file mapping 结构需要实现三个功能：lookups、insertions、deletions。后两者需要 block allocator 提供的物理位置分配功能

##### File Mapping Challenges

以下几个因素导致设计高效的 file mapping 难度较大：

- 碎片：指文件的数据被存储到不连续的物理位置上，会放大某些 file mapping 设计的开销，例如范围树；还会减少顺序访问；在这几时必须考虑碎片，因为去碎片化的开销很大
- 引用局部性：可以记住先前查找的 metadata 数据位置，并且预取下一次查找的位置，来减少 file mapping 的遍历开销，因此在有了高效缓存之后， file mapping 的设计应该针对 random lookups
- 映射结构大小：file mapping 结构也需要占据一部分空间
- 并发性：per-file mapping 结构可以同时发生的操作的种类和分布是有限的。因此在整个 per-file mapping 结构上进行粗粒度的保护即可

##### File Mapping Non-Challenges

- 系统崩溃一致性：文件系统的首要问题是提供一致性，将 metadata 数据结构从一种有效状态转换为另一种有效状态；通常采用某种形式的日志，使得在发生崩溃时仍然能够保持一致性
- 页缓存：传统文件系统在为用户提供服务之前会将读取的数据和 metadata 数据结构缓存在 DRAM 的页中，之后文件系统再批量的写回修改的页，但是这些页缓存需要管理，需要读写来保持一致性，这也会带来开销；对于 PM，OS 管理 DRAM 页缓存不在需要，避开 inode 和目录的页缓存在 PM 优化的文件系统中获得更好的性能

#### PM File Mapping Design

