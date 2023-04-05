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

