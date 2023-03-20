### problem

#### What problem is the paper solving?

由于业务数据爆炸式增长，现代数据中心通常采用联合多个相对较小集群的方案，每个小集群运行一个单独的文件系统实例，整个数据中心采用一个Large-Scale Distributed Filesystem，但这也面临着挑战：
1. 目录树分区，需要同时保证元数据访问负载均衡和局部性
2. 路径解析时延
3. near-root 热点

#### Why is this problem important?
由于现代数据中心业务工作负载特性，文件类操作占比 95.8%，目录类操作占比4.2%，其中readdir操作在目录类操作中占比高达93.3%，目录rename和set_permission仅有0.0083%；

### Context

#### What was the previous state of the art?
采用联合多个相对较小的集群的方案，无法同时保证负载均衡和局部性。

#### How does the paper advanced the state of the art?
实现了元数据访问的负载均衡，同时保证了局部性，并且降低了文件系统的操作时延。

### Design

#### What are the key insight from the design?
1. 目录访问元数据和内容元数据解耦、分区，实现元数据访问负载均衡和局部性
2. 路径解析预测，实现并行目录遍历，降低元数据操作网络传输时延
3. 客户端乐观型访问元数据缓存，解决near-root热点

#### How is the system designed?

1. 将目录访问元数据和内容元数据解耦、分区，实现元数据访问负载均衡和局部性。用 id、name、permission描述目录的访问元数据；使用 entry list、timestamp 描述目录的内容元数据。
	1. 将父目录的内容元数据与子目录的访问元数据、文件元数据划分为一组，将整个目录树分割成相互独立的locality-aware元数据组，实现元数据访问的局部性
	2. 采用细粒度分区策略，基于locality-aware元数据组的目录ID，采用一致性哈希算法，实现元数据操作的负载均衡，同时保证元数据服务扩展时最小化数据迁移
2. 路径解析预测，实现并行目录遍历，降低元数据操作网络传输时延
	1. 对于创建目录操作，通过对birth triple哈希计算，生成目录ID
	2. 对于目录重命名，只需修改目录的访问元数据的key，目录的内容元数据和目录ID保持不变
	3. 对于目录删除操作，删除birth parent时，只删除其访问和内容元数据，保留rename-list。删除重命名目录时，利用back-pointer清理rename-list中的记录
	通过文件系统的语义，保证同一目录下名称的唯一性；在重命名时，通过birth parent目录的rename-list保证名字版本的唯一性；采用cryptographic hash低冲突算法，若发生冲突，在插入目录内容元数据时检测，并通过增加version、renam-list、back-point办法解决
3. 并行解析目录
	1. 预测目录ID，从root开始，以version为0生成所有中间目录项的birth triple，并哈希计算其目录ID，最终得到每个中间目录项的元数据的Key
	2. 并行查找，客户端并行发送所有中间目录项的查找请求，元数据服务端检查目录的权限和预测ID。若预测ID不正确，则终止查找，并返回正确的目录ID
	3. 若中间目录有发生重命名，则需以服务端返回的正确目录ID，对该目录以下的子目录，重复执行步骤1、2
4. 客户端乐观型访问元数据缓存，解决near-root热点
	1. 客户端缓存目录的访问元数据，消除near-root目录解析请求
	2. 按照目录树层级结构组织元数据缓存，并通过LRU策略管理所有的叶子实体（目录，非文件），尽量保证near-root目录元数据常驻缓存
	3. lazily缓存失效，目录重命名和设置权限会使元数据服务器失效，延迟到客户端下次元数据请求时再验证缓存有效性，避免lease机制同时并发更新缓存

### Results

#### How is the design evaluted, what are the key results?

1. 吞吐量：
	1. 由于路径解析和元数据处理，InfiniFS吞吐量满足近线性扩展，但mkdir吞吐量远低于create；
	2. 单个元数据服务器下，InfiniFS文件create性能略低于LocoFS，主要原因在于LocoFS采用供基于hash的KV引擎，但随着元数据服务器数量增多，InfiniFS性能远胜于LocoFS；
	3. InfiniFS性能远胜于HopsFS和Cephfs，其原因在于InfiniFS采用并行可预测目录解析机制
2. 时延：InfiniFS采用了可预测目录解析和乐观访问缓存，因此对于文件create、stat、delete时延比较低，但mkdir时延高于LocoFS，主要由于分布式的事务开销
3. 在加入路径解析预测之后，路径解析耗时降低至Baseline的26%；在加入了客户端缓存之后，解决了near-root热点问题，极大提升了create和mkdir的吞吐量；解耦之后，极大提升了create吞吐量，但mkdir没有明显提升。
4. 在大规模目录树下的性能表现：1000亿量级情况下，能提供稳定的create/stat性能
5. 缓存机制：lazy缓存机制性能优于lease缓存机制；随着缓存大小增加，lazy缓存机制相比lease缓存机制，吞吐量有明显提升
6. 文件重命名时延远低于目录重命名时延，随着服务器增加，文件重命名吞吐量不断增长，目录重命名吞吐量基本稳定
7. 无效预测增加了可预测路径解析的时延，但仍然优于原生路径解析。


#### What related problems are still open?

对于目录的元数据划分是基于父子的，对于祖孙跨服务器的目录解析的局部性似乎不是很友好。


