## boltdb

### boltdb 文件按照页来进行组织，在第 0 个页存放数据库的元数据信息

> 第一个 flag：页面的类型（branch：0x01；leaf：0x02；meta：0x04；freelist：0x10）
>
> bucket root pageid：root page 所在的页编号
>
> freelist：保存所有空闲页面 ID 的页的编号，即索引页的编号
>
> pgid：已分配的页的数量
>
> txid：目前最大的事务 id
>
> checksum：头部字段的校验和

### ![img](jammdb.assets\modb_20211214_e256e17c-5ce6-11ec-b800-fa163eb4f6be.png)

### 硬盘文件中的 B+ 树

#### 叶节点（leaf）

> count：当前页面的 kv 对总数
>
> overflow：若一个页面无法容纳所有的 kv 对，则会分配多个连续的页面，第一个页面的 overflow 字段表示后面连续的 page 数目
>
> flags | pos | ksize | vsize：kv 对的元数据（leafPageElement）
>
> - flags：kv 对的标志位，通常为 0
>
> - pos：kv 对的真实存储位置相对于这个 leafPageElement 元数据的偏移
>
> - ksize | vsize：表示长度
>
> **页面内的 kv 数据按照 key 升序排列**，key 和 value 都是字节数组（SliceParts）

![img](jammdb.assets\modb_20211214_e19e4496-5ce6-11ec-b800-fa163eb4f6be.png)

#### 内部节点（branch）

> 内部节点中只包含 key 数据集合，没有 value。每个 key 是其指向的下一级节点的第一个 key 值
>
> pos | ksize | pgid：key 的元数据（branchPageElement）
>
> - ksize：表示长度
>
> - pos：key 的真实存储位置相对于这个 branchPageElement 元数据的偏移
>
> - pgid：所指向的下一级节点的页面 ID

不管叶子节点还是内部节点，包含的 key 都是按升序排列的，所以查询某个key就非常方便。



查询某个key时，会从根节点一路查询到某个叶子节点。在某个节点内(不管是内部节点还是叶子节点)查询时，采用的算法是二分搜索。

### Bucket & Freelist

#### Bucket

数据库中 kv 对的集合。一个 bucket 中的所有 key 必须唯一，但是不同 bucket 之间的 key 可以重复。

>  Bucket 根据其包含的数据量选择保存方式
>
> - 数据小于或等于 1/4 pagesize：bucket 中的数据较少，存储在一个页上空闲空间较大，因此可以直接内联保存在当前页面中；在叶节点的布局前加上了 bucket header；leafPageElement 中的 flags 字段不再是 0，而是 0x01；key value data：包含 bucket 中所有的 kv 对数据；
>
>   ![img](jammdb.assets\modb_20211214_e07e2086-5ce6-11ec-b800-fa163eb4f6be.png)
>
>   ```go
>   type bucket struct {
>     root     pgid   // 内联的 bucket 实际所在的页面 ID
>     sequence uint64 // monotonically incrementing, used by NextSequence()
>   }
>   ```
>
>   
>
> - 超过 1/4 pagesize：为 bucket 中的数据分配单独的 page

#### Freelist

空闲页面ID列表

> count：空闲页面的数量
>
> overflow：如果空闲页面很多，一个页面保存不下，就会申请几个连续的页面来保存；这时就要用到 overflow 字段

![img](jammdb.assets\modb_20211214_e0a0a818-5ce6-11ec-b800-fa163eb4f6be.png)

### 事务（Transaction）

只读事务，可同时执行多个只读事务，共享锁

读写事务，任何时刻只能有一个读写事务，排他锁

整个 DB 有一个读写锁

#### COW（Copy-on-write）

在事务的执行期间，不管对数据做过什么修改，最后提交事务(commit)时，修改的数据才会持久到存储介质。这就意味着，在提交这个时间点之前的所有操作，都是在内存中进行，对持久存储介质上的数据没有任何影响。

#### 事务提交（commit）

内存中的 node 对应到文件中的 page，每一个 node 最终会保存到文件中的一个新的位置(page)，所以会释放 node 之前所在的 page ID，并申请一个新的 page ID。这么做是非常巧妙的一个设计，是保证事务原子性的第一个关键点。

在提交当前事务时，页面会重新分配，但是旧的页面不会马上进入 freelist，而是处于 pending 的状态，要等到下一次事务才会被释放，通过这一点保证事务原子性。

交替使用两个 metadata page，保证原子性。



