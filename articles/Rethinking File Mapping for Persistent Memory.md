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

##### Traditional，Per-File Mapping

传统文件系统中每个文件都有一个 file mapping 结构，可以伸缩，保存在一个隔离的固定大小的 inode table 里面，通过文件的 inode number 来索引，具有较高的空间局部性，单个文件的映射都集中在一个结构中（可以直接缓存在内存中，相邻的逻辑块在 file mapping 结构中的映射也相邻），并发访问会存在问题，因为限制了访问的模式，物理块分配单独分配，使用的是用红黑树实现的块分配器，per-file mapping 要求必须支持 resizing，但这个操作的开销很大（有多层的间接索引，每次映射操作需要多次的内存引用）

- Extent Trees（B-trees contain extents），在 linux 内核上进行了测试，设计了直接操作 PM 的范围树。每个范围条目设计了验证位，可读的情况下才会进行复位，来保证并发操作，其他复杂操作的一致性通过 undo log 来提供，在 DRAM 中保存上一次访问路径的 cursor，改善局部性访问，但是需要多次内存访问，树中的一层需要多次内存访问（二分搜索）才可以找到下一层的位置
- Radix Trees（基数树），类似于页表机制，在 DRAM 中保存树中上一次搜索的位置，并且通过 CPU 的原子指令保证一致性，由 undo log 来保证 resizing 一致性，不够紧致，遍历花费的时间更长，但是 mapping 操作非常简单，树中的每一层只需要一次内存访问

##### PM-specific Global File Mapping（怎么保持一致性）

设计了全局的 file mapping（hash table），而不是针对每个文件，因此不再需要 inode 表，利用 hash 表内存访问少，同时避免了因为每个文件大小伸缩而产生的 resizing 开销，因为文件系统创建时，物理块的数目就已经知道，因此可以静态的创建，并且不会受到碎片的影响，由于扁平寻址方式，并发访问下的一致性通常可以基于每个 hash 桶进行解析。

在 DRAM 中使用固定大小的转换关系（逻辑块到物理块）缓存来加速局部查找，使用的索引和全局 hash 表一致。与页缓存不同，页缓存是缓存 mapping 结构。由于 hash 表的随机性，表现出较低的局部性，全局 hash 表比 per-file hash 表表现的局部性更低，转换关系的缓存改善了这种情况，加速了局部访问。

- Global Cuckoo Hash Table：使用两个不同的 hash 函数 hash 两次，查找只需要访问两个位置，插入时如果两个位置都满，则会挑出一个已存在的 entry，将其移动到备用位置，继续上述过程，直到不再发生冲突。（占了位置就踢出去）
  - 连续数组，创建文件系统时静态分配，<u>在保持一致性时，先持久化 mapping info（物理块号，大小），再持久化 key（inum，逻辑块号），用 key 作为有效指标。（不太明白什么意思）</u>插入时保持一致性很复杂（会移动之前的 entry），通过在 undo log 中记录操作来保证。复杂的更新需要的原子操作也更多，因此使用 intel TSX 代替 per-entry 锁提供隔离（非常有必须，插入可能会跨文件同时发生）
  - 一对一映射，对于连续分配的块不友好，因此每个 entry 都包括一个字段来记录连续的块数目。每个 entry 还包括一个反向索引，保证在删除末尾的块时同一个连续组的所有 entry 会更新。（感觉开销很大？？？）
  - 可以并行查找，使用 SIMD 指令
  - 在 cuckoo 策略和线性探测之间需要权衡，cuckoo 只需要查找两次位置，但是局部性很差
- HashFS：为了减少插入操作的开销，hash 表 + 块分配器
  - 将设备划分为 metadata 区域和 file data 区域，没有范围表示，只提供逻辑块与物理块的一对一映射，将 inum 与 逻辑块号整合成 64 位的整型，用原子指令可以保证一致性，
  - 使用 SIMD 指令进行向量 hash  操作，如果没用到最大的 SIMD 带宽，则会预取 entry 缓存进入转换关系缓存中
  - 使用线性探测，局部性较好
  - 不会像 cuckoo 把 entry 踢出去再重新分配，不会破坏之前的 entry 的映射关系
  - 只用一次 hash，但是解决冲突的开销很大，在负载因子较高的时候性能较差

#### Evaluation

- 局部性：cold cache 上快 15 倍，因为对 tree 的操作需要多次的间接访问，cold cache insert 时，Cuckoo hash 策略的全局 hash 表表现的性能差（因为 cache 未命中，并且块分配使用的是红黑树，并且在内核线程之间还有互斥锁）。全局 hash 表比 per-file 映射结构更高效，尤其是访问没有局部性时
- 碎片：只有 extent tree 会受到较大的影响，HashFS 不受到碎片的影响
- 文件大小：per-file mapping 会根据文件大小拓扑结构不同，global hash table 则是扁平的，只有 Cuckoo hash 策略在插入的开销较大
- IO size：随着 IO size 增加，延迟会增加（不管是什么 file mapping，均会增加，但是增加的程度不同）
- 空间利用率：global hash 表随着已经使用的 bucket 增加，会导致碰撞增加，从而会导致 file mapping 延迟增加；per-file mapping 延迟不会因为空间利用而变化，80% 空间利用率时，HashFS 和 Cuckoo hash table 区别不大，在 95% 时，HashFS 读的延迟较高，插入的延迟一致。
- 并发：隔离机制并不是瓶颈，并且 global file mapping 可以有效的隐藏同步开销
- 页缓存：提倡使用开销较低的缓存方式（cursor），而不是页缓存

##### File Mapping via Level Hashing？

不建议使用 PM storage structure



点评：

- 设计了 4 种在 PM 设备上的 file mapping 数据结构，进行了测试，说明了传统的 per-file mapping 数据结构、以及页缓存机制不适用于 PM 设备
- 对于并发时的一致性，依赖 SIMD 指令以及某一项技术，并且文章没有详细的说明采取什么样的策略，只简单提了一下
- 这里的 PM 设备是用在中间层，还是整个底层的块设备都是用的这个，这个没有说清楚

![](Rethinking File Mapping for Persistent Memory.assets\存储结构图.png)

