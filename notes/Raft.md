To solve the problem brought by the single master, Raft protocal select a leader from followers.


#### Problem
##### The Split Brain Problem
- the most straightforward way: a single coordinator
	- exist an availability problem
	- primary-backup architecture will bring the split brain problem
- 


#### design
所有节点自动时，默认为 follower 状态，在一段时间没有收到 leader 的 heartbeat 之后，从 follower 切换为 candidate 状态，从而发起选举。如果收到 majority 的票，则切换到 leader 状态，若发现其他节点比自己更新，则主动切换到 follower。

leader 与 follower 以相同的顺序执行请求（复制状态机），使用 replicated log 保证所有节点能够获得相同输入，并以相同顺序执行。

leader 将请求封装成 log entry，复制到所有的 follower 节点。

当出现 leader 与 follower 不一致的情况之后，leader 强制 follower 复制自己的 log。

leader 维护了 nextIndex 数组，记录了 leader 可以发送给 follower 的 log index，初始化为 leader 的最后一个 log index + 1


### A

#### 提示：
-  论文图 2，关注发送和接收 RequestVote rpc，与选举有关的服务器规则，以及与选举有关的状态
- 在Raft .go中，将图2中领导人选举的状态添加到Raft结构中。需要定义一个结构来保存关于每个日志条目的信息。
- 填写RequestVoteArgs和RequestVoteReply结构体。修改Make()以创建一个后台go协程，当它在一段时间内没有收到另一个对等点的消息时，通过发送RequestVote rpc定期启动leader选举。这样，如果已经有领导者了，同伴就会知道谁是领导者，或者自己成为领导者。实现RequestVote() RPC处理程序，这样服务器就可以互相投票。
- 要实现heartbeats，定义AppendEntries RPC结构体(尽管您可能还不需要所有的参数)，并让leader定期发送它们。编写一个AppendEntries RPC处理程序方法用于重置选举超时，这样当一个服务器已经当选时，其他服务器就不会站出来担任领导者。
- 确保不同对等点的选举超时不总是同时触发，否则所有对等点只会投票给自己，没有人会成为领导者。
- 测试要求领导者发送心跳rpc每秒不超过10次。
- 测试要求你的Raft在旧领导者失败的5秒内选出一个新的领导者(如果大多数同行仍然可以沟通)。然而，请记住，在分裂投票的情况下，领导者选举可能需要多轮(如果数据包丢失或候选人不幸地选择了相同的随机退选时间，就会发生分裂投票)。您必须选择足够短的选举超时(以及心跳间隔)，以便即使需要多轮投票，选举也很可能在不到5秒的时间内完成。
- 论文的第5.2节提到了150到300毫秒范围内的选举超时。只有当领导者每150毫秒发送一次心跳(例如，每10毫秒一次)时，这样的范围才有意义。因为测试者限制你每秒几十次心跳，你将不得不使用比纸上的150到300毫秒更大的选举超时，但也不能太大，因为那样你可能无法在5秒内选出领导人。
- 编写定期或在时间延迟后执行操作的代码。要做到这一点，最简单的方法是创建一个带有调用time.Sleep()循环的goroutine;参见Make()为此目的创建的ticker() gor例程。不使用 go 的 time.timer 或 time.ticker。
- 实现 getstate
- 测试人员在永久关闭一个实例时调用Raft的rf.Kill()。你可以使用rf.killed()检查Kill()是否被调用。您可能希望在所有循环中都这样做，以避免 dead raft 实例打印令人困惑的消息


- follower 只会回答 leader 和 candidates 的请求，如果 client 与 follower 进行通信，follower 则会转发给 leader。
- 将时间划分为 term，每个 term 从选举开始，如果选举不成功，重新开始投票，进入下一轮 term。
- 每个 server 记录一个单调递增的 term，当 server 之间通信时，会交换 term 信息，较小 term 的 server 会更新自己的 term 为较大的
	- 如果 candidate 或者 leader 发现自己的 term 过期，则理解转变为 follower
	- 如果 server 收到过期的 term，直接拒绝请求
- 如果 follower 一段时间没有接收到来自 leader 的 heartbeat，则会开始新一轮的选举
	- follower 增加自己的 term，并转变为 candidate 状态，之后给自己投票，并且并行的向其他的 server 发送 RequestVote RPC，candidate 如果获得大部分的投票，则成为 leader，并发送 heartbeat，通知其他 server
- 如果 candidate 在等待投票结果时，收到其他自称为 leader 的 RPC，如果这个 RPC 请求的 term 至少和 candidate 一样大，那么 candidate 会转变成 follower，如果 RPC 请求的 term 比 candidate 小，candidate 直接拒绝 RPC，并继续保持 candidate 状态
- 如果没有获胜者，则会重新开始新一轮选举，通过 随即超时机制 保证重新选举出现的概率非常小，

- leader 在收到 client 的请求之后，，在 log 后增加一个 entry，之后通过 AppendEntries RPC 并行的向其他的 server 发送这个 entry 的副本，leader 在收到安全备份的回复之后，向 client 发送结果。（即使已经回复客户端了，leader 仍然会向那些没有收到回复的 server 发送 AppendEntries RPC）
- 一旦 leader 创建的 log entry 被大多数 server 备份，则这个 log entry 将会被提交（包括之前的 leader 创建的）
- leader 跟踪它自己已知的被提交的 log entry 的最高索引，并且会将这个索引附加在 AppendEntries RPC 请求中，让其他的 server 知道，一旦 follower 知道某个 entry 被提交了，它会应用到本地的状态机

log 机制的两个特点
- 如果 term 和 index 相同，则 log 中的指令相同
- 如果 log term 和 index 相同，则这之前的 log 都相同（由 AppendEntries RPC 的简单一致性检查保证，leader 会把这一项 log 之前紧挨着的 log 也发送出去，如果 follower 不能找到前一个 log 和 term时，会直接拒绝）

如果日志出现不一致，leader 会强制让 follower 复制自己的日志。
- leader 为每个 follower 建立一个 nextIndex，记录 leader 将会发送给 follower 的下一个 log entry 的索引，leader 初次掌权时，将所有nextIndex值初始化到其日志中最后一个索引之后的索引
- 如果 follower 日志不一致，则在 AppendEntries RPC 检查时会失败，在被拒绝之后，leader 会递减对应的 nextIndex 值，并继续 AppendEntries RPC 检查，直到日志匹配。follower 将会丢弃那些冲突的 log entry，并添加 leader 的日志。（可以针对每个 term 进行优化，而不是每条 log entry）

安全（限制了哪些 server 可以被选为 leader，确保 leader 包含之前所有提交的 log entry）
- candidate 必须和集群中的大多数 server 取得联系（意味着提交的 log entry 必存在与某个 server 中）
- RequestVote RPC包括了 candidate 的信息，如果投票者的日志比 candidate 的更新，则会拒绝投票（term 较大的 log 是较新的，如果 term 相同，则较长的 log 较新）
- 不通过统计计数来提交前一个 term 的 log entry，通过计算副本，只提交 leader 当前 term 的 log entry（因为之前的两条属性，简介保证了之前的 log entry 被提交了）

log matching 属性保证了后来的 leader 必定包含之前的 leader 提交的 log entry


更换配置
两阶段， 第一阶段，集群先切换到过渡阶段，称为联合共识，一旦联合共识被提交，系统再切换到新的配置。
- 联合共识结合了新旧配置
	- 日志条目将复制到两种配置下的所有服务器
	- 来自任意配置的 server 都能成为 leader
	- 使用特殊的 log entry 存储配置信息
- 当 leader 收到从 Cold 转变成 Cnew 的配置信息时，它将其作为 log entry 使用之前的机制进行备份，一旦 server 添加了这个 configuration log entry，它之后就会使用这个配置（无论是否被提交）
- leader 将会使用 Coldnew 来决定 Coldnew log entry 是否被提交，如果 leader 故障了，则新的 leader（只有 Cold 和 Coldnew 这两种状态） 会被选出，Cnew 不能单边做出决定
- 一旦 Coldnew 被提交，则 old 和 new 均不能单独做出决定，只有带了 Coldnew log entry 的 server 能够被选为 leader
- 之后就可以在集群中对 Cnew 进行备份（Cnew 也会立即生效）

重新配置需要注意的问题：
- 新的 server 没有初始化 log entry，在配置改变之前引入新的阶段，这个阶段，新的 server 作为不能投票的成员加入集群（在统计数量时不计算，但是 leader 会复制 log entry 副本），一旦新的 server 跟上了集群的进度，则可以重新配置
- 第二个问题：集群的 leader 可能不属于新的配置，此时，leader 一旦提交了 Cnew（leader 只会统计除自己外的其他 server 的数量），则直接返回到 follower 状态
- 第三个问题：那些被删除的 server 能中断集群，这些服务器收不到 heartbeat，因此会开始新一轮的选举，他们会以新的 term 发送 RequestVote RPC，从而导致当前的 leader 转变成 follower 状态（但会循环产生这种情况）。因此，当 server 认为 leader 存在时，会直接拒接 RequestVote RPC（即 server 收到了 leader 的 heartbeat 消息之后，生成的随机超时范围内，不会改变投票，不会更新 term）