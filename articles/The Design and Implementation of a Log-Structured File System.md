[TOC]

## The Design and Implementation of a Log-Structured File System

#### Abstrac：

- 所有文件的修改顺序的添加在 disk 的后面，加速了文件的写以及恢复（消除了所有的寻道）
- disk 上只有唯一的一种数据结构：log，其中包含了索引信息来高效的从 log 中读取数据
- 性能：在小文件写比 UNIX 高一个数量级，在读以及大规模写与 UNIX 相当甚至更优

#### Introduction：

- 现状：CPU 速度不断增加，硬盘速度却增长的缓慢，硬盘读写速度成为大多数应用的瓶颈
- 为了高效，必须保证在写的时候总有足够大的空间，因此将空闲空间划分为 segment，程序 segment cleaner 周期性的压缩严重碎片化的片段来重新生成大的 segment
- segment cleaner 采取的策略：将旧的、变化慢的数据与新的、极速变化的数据区别对待
- 性能：65 ~ 75% 的带宽写数据，剩下的带宽用于 clean（UNIX：5 ~ 10% 的带宽写，剩下的寻道）



#### Desing for file systems of the 1990's

介绍 **技术**以及**任务量**给设计文件系统带来的影响

##### Technology

- 三个重要组件：处理器、硬盘、内存；处理器的速度加快给计算机其他部件的发展带来了压力；硬盘的改进更多的是容量和开销，而不是性能（传输带宽有改进（磁盘阵列以及并联磁头磁盘）、访问时间基本上没有改进）；现代文件系统在内存中缓存了最近的文件数据，内存增大，可以缓存的文件数据更大，通过吸收读取请求可以减少磁盘的工作量，但是最终必须要在磁盘上写入，因此磁盘的性能主要取决于写的性能；大文件缓存可以充当写入的缓冲区，在将任何缓冲区的数据写入磁盘之前可以收集大量修改后的块，使得写数据块更加高效（只需一次寻道），但是也会造成崩溃时丢失的数据更多

##### Workloads

文件系统设计最难有效处理的工作负载是在办公软件核工程软件环境中，这个环境主要取决于访问小文件（小文件经常会产生随机磁盘I/O，创建和删除这些小文件取决于更新文件系统的元数据）

##### Problems with existing file systems

- 将信息分散到磁盘的各个位置，需要多次寻道
- 写数据采取同步的方式，即使 UNIX FFS 写数据块是异步的，但是对于文件系统的元数据的写仍然是同步的

#### Log-structured file systems

将许多的小规模的同步的随机访问转化为大规模的异步顺序传输

关键点：

- 如何从 log 中取回信息
- 如何管理空闲空间，保证写新数据时总有足够大的空间

##### File location and reading

- 在 log 结构中输出索引结构信息，允许随机访问检索
- 一旦找到 inode，LFS 和 UNIX FFS 读文件需要的磁盘 I/O 请求次数相同
- 不把 inode 放在固定位置，而是存放在 log 中，在 log 中维护一个 inode map 数据结构来获得每个 inode 的位置
- inode map 写到 log 中的块上，在一个固定检查区域中包括了所有 inode map 所在的块

##### Free space management: segments

- LFS 将 disk 划分为固定大小的段，段总是从头写到尾，段需要重写时，会复制出里面的活跃数据（段大小通常为 512KB 或 1MB）

##### Segment cleaning mechanism

- 读取多个片段的数据到内存中，识别出活跃的数据，并将它们写回到较少的段中
- segment clean 程序需要识别出一个段中哪些块是活跃的，还需要知道这个块属于哪个文件以及在文件中的位置
- 通过在每个段中存放段摘要信息来解决上述问题（例如：对于每个文件数据块，段摘要信息中包含了所属的文件编号以及块编号）
- LFS 能够通过段摘要信息区分出活跃信息，一旦识别了某个块，那么就可以通过检查所属文件的 inode 或者 间接块上是否有合适的块指针指向这个块这种方式，判断出这个块是否活跃
- LFS简化了活跃性检查，在每个文件的 inode map 实体中记录了版本号，版本号与 inode 编号形式相结合，构成文件内容的唯一标识符 uid，段摘要信息记录了段内每个块的 uid，当块的 uid 与当前 inode map 中保存的不一致，就会直接舍弃这个块
- 删除了 bitmap 和 free_list，减少这部分数据维护和恢复的开销

##### Segment cleaning policies

四个问题：

- 什么时候运行 segment cleaner 程序？以较低的优先级在后台持续运行；只在晚上运行；只在磁盘空间耗尽时
- 一次清理多少个段？一次清理的段的数量越多，段重新安排的机会越多；
- 哪些段需要被清理？明显的结果是碎片化最严重的段，但实际上，这并不是最好的选择
- 当活跃的块被写出时，应该怎么组织？一种方式是增强局部性，方便之后的读操作；另一种方式是根据块最后一次修改的时间排序，将 age 相似的块组织到同一个段中（age sort）

对于第二个问题，LFS 当干净的段数量低于某个阈值时开始清理，直至干净的段高于另一个阈值，LFS 的性能与阈值的设置不敏感；相比之下，第三、四个问题的策略更为重要。

写开销：磁盘写新数据所占的带宽与最大带宽的比值的倒数（LFS 中磁盘寻道和旋转的时间可以忽略不计，u 表示段的利用率）
$$
write cost = \frac{total\ bytes\ read\ and\ write}{new\ data\ written}
			= \frac{read\ segs\ +\ write\ live\ +\ write\ new}{new\ data\ written}
			= \frac{2}{1 - u}
$$

##### Simulation results

uniform 策略：每个文件在每个阶段被选择的可能性相同

hot-and-code 策略：10% 的文件访问次数占90%，90% 的文件访问次数只占 10%；hot 和 cold 组内部被选择的概率相同

模拟测试结果：

- 使用的 uniform 策略，写开销也比上述公式预测的结果要低很多
- LFS hot-and-code 策略，曲线和 uniform 策略基本相同，但是活跃块在写出时是按照 age 来进行排序，因此会导致长期活跃的块与短期活跃块相分离
- hot-and-cold 策略局部性更强，性能反而更差（贪心策略选择碎片化最严重的段，但是 cold 组内的段碎片化的程度增长的很缓慢，因此得不到及时清理，导致大量的空闲空间浪费）
    - 解决措施：区别对待 hot 和 cold 组，cold 组的段的空闲空间更有价值，一旦 cold 组内的段被清理了，就需要花很长一段时间来收集无法使用的空闲空间，hot 组内的段清理的价值更低，因此 hot 组的清理可以推迟一些

- cost-benefig 策略：选出 benefit/cost 最高的段进行清理，采取这种策略，随着局部性提高，性能越好（u 表示段利用率，age 表示最近修改的文件的时间到当前时刻的时间）
    $$
    \frac{benefit}{cost} = \frac{free\ space\ generated\ *\ age\ of\ data }{cost} = \frac{(1-u)*age}{1+u}
    $$

#####  Segment usage table

LFS 维护了一个 segment usage table 的数据结构来支持 cost-benefit 策略

- 对于每个段，这个表记录了段中的活跃字节数以及段中的每个块最近修改的时间，segment cleaner 根据这两个值来选择段（这个segment usage table 所在的块在 log 中记录，并且被存放在固定检查区域）

#### Crash recovery

一旦发生系统崩溃，磁盘上最近的几次操作会处于不正常的状态，UNIX FFS 没有 log 结构，因此必须要扫描整个磁盘来恢复，但是 LFS 只需要恢复最近的几次操作即可，这些操作被记录在 log 的最后位置，因此可以快速恢复（很多数据库系统以及文件系统都采用了这种方式），LFS 采用了 checkpoints 以及 roll-forward 两种结合的恢复方法

##### Checkpoints

checkpoint 是 log 结构中所有文件系统结构一致性和完整性的一个位置。LFS 用两个阶段性程序来创建一个 checkpoint。

- 第一阶段记录了所有的修改信息到 log 结构中，包括文件数据块、间接块、inode 以及 inode map、segment usage table 所在的块。
- 第二阶段会将 checkpoint 区域记录到磁盘上的固定位置

在恢复时，LFS 先读取checkpoint 区域并且用其中的信息恢复内存中的数据结构，为了避免 checkpoint 崩溃，因此进行了备份。并且在区域的最后位置会记录 checkpoint 时间，如果checkpoint 不成功，这个时间不会更新。

LFS 会以周期性间隔在文件系统被卸载或者系统关闭时进行 checkpoint。时间间隔越长，虽然减少了检查的开销，但是每次恢复需要回滚的时间越多，时间间隔越短，增加了检查开销，但是恢复所需的时间也越短。

通过写入给定量的新数据之后进行周期性的检查，从而避免了恢复时间过长，也减少了检查花的时间

##### Roll-forward

LFS 不仅仅是恢复最后一次检查之前的数据，也会扫描在 log 中在最后一次检查之后重新写入数据的段，尽可能恢复多的数据。

- 使用段摘要块中的信息来恢复最近写的文件数据，如果摘要信息显示创建了一个新的 inode，则会读取 checkpoint ，并且更新 inode map。
- roll-forward 程序也是根据从 checkpoint 读取的 segment usage table 中的利用率来变化，在 roll-forward 之后需要记录活跃数据，以及文件删除的数据
- LFS 会在 log 中记录每一个目录的变化的数据结构（directory operation log），用来恢复目录和 inode 的一致性，这个目录变化的数据结构会记录 create、link、rename、unlink 等操作，会记录目录实体的位置，以及 inode 的引用计数
- 恢复程序会将恢复的信息的变化附加在 log 中，并在一个新的检查区域中记录

唯一不能恢复的操作是创建了一个新文件，但是 inode 从没有写过，这种情况下，目录实体会被删除

directory operation log 以及 checkpoints 操作之间的交互，给 LFS 带来了额外的同步问题

#### Experience with the Sprite LFS

- 其余的部分已经实际使用了，只有在 roll-forward 还没有用在生产系统中，生产系统的磁盘采取短间隔（30s），并且将最后一次检查之后的信息全部抛弃

- LFS 比 UNIX FFS 的复杂度更低，尽管 segment cleaner 增加了复杂性，但是这与减少的 bitmap 以及 UNIX FFS 要求的布局策略的复杂性相抵消；但是恢复操作，LFS 比起 UNIX FFS 扫描整个磁盘降低了不少复杂性
- LFS 比 SunOS 在小文件的创建访问方面快，大文件的顺序/随机写比 SunOS 快，但是在顺序/随机读方面比 SunOS 稍微逊色
- 传统的文件系统假设读取是符合局部性的，因此会在写的时候会为了未来的读产生额外的开销来提高局部性；LFS 则是考虑的短期局部性，会将最近修改的文件放在一起
- 在实际的运行时，考虑写开销时，LFS 比预期的更好

#### Related work

#### Conclusion



#### 阅读中产生的想法：

利用 log 来异步？？？

在内存的缓冲区中记录 log ？？？

