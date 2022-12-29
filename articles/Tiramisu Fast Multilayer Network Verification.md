

[TOC]

## Tiramisu: Fast Multilayer Network Verification

**网络验证：**基于形式化方法验证了网络行为与网络意图之间的一致性。使用严格的数学方法,能够基于网络的配置或转发状态推断出所有可能的网络行为,然后将网络 行为与意图进行比较,验证网络意图是否得到了正确实现。网络验证的研究意义在于,可以自动化且准确全面地发现网络中潜在错误以及错误原因,避免了繁琐易出错的手动分析

数据平面验证、控制平面验证、有状态网络控制

网络背景知识：

- 从网络运行流程上看,网络管理人员首先通过管理平面使用策略实现网络意图,并下发到控制平面,然后控制平面结合网络拓扑等环境信息生成数据平面,最后数据包根据数据平面进行转发。
- 数据平面验证通过输入数据平面信息（例如转发表或流表）和 网络拓扑,验证数据平面下所有数据包转发行为与网络意图的一致性
- 控制平面验证通过输入控制平面信息 （例如配置或 SDN 程序）、网络拓扑与其他环境信息（例如链路状态、路由通告）,验证在这些信息生成的数 据平面下,所有的数据包转发行为与网络意图的一致性
- 目前,数据平面验证主要验证数据包转发相关的网络属性,如可 达性、无转发循环、区域流量隔离。数据平面验证需要实现实时验证,通过持续采集与验证数据平面信息,保证数据平面行为与策 略之间的实时一致性
- 控制平面验证可以验证可达性、无转发循环等数 据包转发相关的网络属性；此外还可以验证假设网络环境下的网络属性验证。例如,保证在 任意 k 条链路失效的情况下,特定网络设备之间的可达性始终能够成立。

**控制平面验证：**可以在网络配置部署之前完成验证。可以直接分析配置文件，更有效定位错误配置。此外，控制平面在某些情况下存在多个收敛数据平面，且其最终收敛到的数据平面是不确定的。

- 传统网络中的控制平面验证通过分析分布在各处网络设备中的配置文件验证网络策 略
- SDN 中的控制平面验证则通过分析控制器或应用中的程序验证网络策略

**Tiramisu：**第一层图模型用于建模参与路由信息传播的路由进程,第二层图模型在第一层的基础上进行更加细粒度的建模,以建模各 种协议交互行为

### Abstract

分布式网络控制复杂，第 2、3 层多种协议交互，导致网络配置复杂且容易出错。state-of-the-art 检查工具要么是违反了密钥的属性，要么就是速度太慢，或者就是无法对网络中的要素进行建模。

提出了新的通用的多层图控制模型，提供快速、定制属性的验证算法。Tiramisu 能够验证各种实际的和综合的配置是否出错，在小型网络中延迟 < 0.08s，在大型网络中延迟 < 2.2s，比 state-of-the-art 快 2600倍，并且不损失通用性。

### Introduction

许多的网络使用复杂的拓扑结构和分布式控制来实现复杂的目标。在拓扑级别上，网络采用技术将多个链路虚拟化成为逻辑上相互隔离的广播域。网络中的 Control Plane 则是通过使用多种路由协议，与其他的机器通过复杂的方式交换路由信息。这方面的虚拟技术也很常见（例如虚拟化路由和转发）。

在这些协议所需要的详细配置中，很容易产生 bug。很多情况下，网络发生错误，控制收敛到新的路径时，这些 bug 就会被触发，会导致灾难性的后果：

- 可达性黑洞
- 受限制的服务访问可能会变得完全开放
- 可能会导致选择开销较大的路径

有很多的工具试图检验网络是否违反了某个重要策略。

- control plane analyzers 会主动验证网络是否满足针对各个环境下的策略
- ARC（一种基于图算法的工具，State-of-the-art）会把网络中可能出现的所有路径建模成一系列加权有向图
- Minesweeper（基于 SMT 的工具），通过使用逻辑约束/变量来对路由交换信息、路由选择和故障进行编码，完成对 Control Plane 的建模
- Plankton（基于 ESMC 的工具），使用自定义的语言对路由协议进行建模，以便 ESMC 可以探索由 Control Plane 执行产生的许多的 Data Plane 状态

这些 Control Plane 工具性能和通用性很差。ARC 抽象了很多低级别的 Control Plane 细节，使得能够利用多项式时间复杂度的图算法来进行验证，为所有的工具提供好的性能。但是这种抽象忽视了许多网络设计结构，包括 BGP 属性以及 iBGP。虽然其他的工具检查了这些细节，能够在低级别进行建模，但是性能极差，尤其是探索故障时。

所有的工具都忽略了 VLAN 和 VRF。

文章寻求一种快速的通用的 Control Plane 验证工具，同时还考虑 2.5 层的协议，例如 VLAN。

文章的团队发现在性能和通用性之间的权衡在某种程度上是人为的，源于 Control Plane 编码和使用的验证算法的不自然耦合。

Tiramisu 的关键是将编码与验证算法解耦。

- 使用了新的、充足的编码技术，能够对 Control Plane 特征和网络设计结构建模。并且这个编码可以让 Tiramisu 可以针对不同类型的政策自定义验证算法，比 state-of-the-art 大幅提高了性能。
- Tiramisu 的网络模型使用图作为基础，类似于 ARC，但是图模型是多层的，使用了多属性边权重，从而捕获协议之间以及虚拟和物理链路之间的依赖关系，并考虑了过滤器、标签和协议成本/首选项等。
- 对于自定义验证，Tiramisu 指出大部分政策可以分成三类，Tiramisu 利用了底层的图模型结构为每个类型开发高性能的验证算法
  - 需要给出故障情况下网络的实际路径的策略
  - 关心路径可能表现出的某些定量属性的策略
  - 仅仅关注路径是否存在的策略
- 对于策略类别1，使用 Tiramisu Path Vector Protocol（TPVP）高效的解决稳定路径问题。
  - TPVP 模拟 Control Plane 计算跨多个设备和在 Tiramisu 网络模型上运行相互依赖的协议
  - TPVP 的域特定路径计算比 SMT 快，在验证类别策略时快几个数量级
  - TPVP 在 Tiramisu 富图上使用路由代数允许它一次性计算路径，而 ESMC 则模拟协议，并探测状态，考虑到协议之间的依赖关系一次计算一个

- 对于策略类别2，Tiramisu 使用整数线性规划公式，仅仅对正在验证的策略相关的变量进行建模，比 SMT 和 ESMC 快，因为这两个都需要探测更多的变量
- 对于策略类别3，Tiramisu 使用新的图遍历算法来检测路径是否存在，在多个 Tiramisu 子图上调用规范的深度优先搜索，并考虑路径上使用控制路径存在的标签，Tiramisu 能够达到 ARC 的性能，却不失通用性
- 对于大多数类别1和2的策略，Tiramisu 利用底层的图结构模型，发展出一种基于图算法的加速器来加速验证。使用多种动态规划算法来计算 k 最短路径问题来减少不必要的路径（与故障场景无关的路径）探测

用 Java 实现了 Tiramisu，并且在许多真实、复杂配置、横跨校园、ISP和数据中心网络中进行了测量，比 state-of-the-art 性能好，更通用，并且考虑到了第2/3层的特性（Minesweeper 和 Plankton 没有考虑）。针对不同类别的策略，Tiramisu 的延迟为 60、80、3ms。比 Minesweeper 快 80x、50x、600x。在iBGP 网络中，Tiramisu 比 Plankton 快 300x。Tiramisu 的 TYEN 加速器将性能提高了 1.3~3.8 倍，对于具有 ~ 160 个路由器的网络，对每个流的验证结果延迟为 ~100ms

### Motivation

#### Existing control plane verifiers

state-of-the-art 基于三个不同的技术：图算法、符号模型检查、显式状态模型检查

- **图算法：ARC** 对于网络层 Control Plane 用有向图建模，对网络转发行为（即从源子网到目的子网发送包）进行编码。顶点表示路由进程（在特定设备上运行路由协议），有向边表示能够允许路由进程之间交换公告的可能网络跃点，边权重表示 OSPF 的开销或者是 AS 的路径长度。ARC 只检查简单的图属性，利用图算法，ARC 比其他的基于模拟的验证算法性能高几个数量级。但是这种简单的图模型没有考虑第2、3层的许多协议特性
- **符号模型检查：Minesweeper** 通过建模和解决 SMT 问题来验证网络的策略合规性。SMT 约束对网络的 Control Plane 和 data Plane 行为（M）以及目标策略取非（!p）进行编码，如果不满足 M 且 !p，则策略是满足的。为了覆盖广泛的网络设计，Minesweeper 会使用很多变量，因此搜索空间很大，并且通过这一个通用的 SMT 模型来验证所有的策略。为了验证 k 错误，Minesweeper 可能会在最坏的情况下枚举 k 链路故障的所有可能组合。
- **显式状态模型检查： Plankton** 用路径向量协议（PVP）通过 Promela 语言对不同的 Control Plane 协议建模。它根据数据包等效类（PECs）跟踪 Control Plane 中定义的协议之间的依赖关系，再通过 SPIN（显式状态模型检查器）搜索所有的状态，发现违反策略的状态。Plankton 并行使用了多个 SPIN 来加速这个检查过程，还是用了其他的优化方法，但是还是需要枚举出许多失败场景下的状态空间，再顺序检查

#### Challenges

- **跨域层次的依赖关系**：eBGP、iBGP、VLAN、OSPF 这些协议处于不同的层次，路由器承担多个角色。ARC 简化了图的抽象，不对 iBGP 建模，因此不能对 iBGP-OSPF 依赖关系进行建模；Minesweeper 尽管可以对 iBGP 建模，但是非常低效；Plankton 将 iBGP-OSPF 依赖编码成 PECs 之间的依赖，因此不能使用 SPIN 并行处理，速度慢。这种跨越层次的依赖关系在其他网络中也很常见。必须要全部考虑这些依赖关系，并且正确、高效的建模。
- **协议属性**：新增了协议的属性或者节点，不能考虑进去
- **错误**：有些错误会被误判，Minesweeper 和 Plankton 枚举很多错误场景来顺序检查，速度很慢。

这些错误都没有考虑策略。

### Overview

综合考虑已经存在的检验工具的缺点，Tiramisu 基于富图网络模型，捕获考虑了跨越多个层次的转发和失败依赖关系。图对于协议来说是非常自然的建模工具。

#### 基于图的网络模型

Tiramisu 构建了两种相互关联的图：路由邻接图（RAGs）、流量传播图（TPGs），前者能够确定哪些路由进程可以学习到特定目的的路由，后者更加详细，考虑到所有的先决条件。验证算法是运行在 TPGs 上的。

- RAGs：包含每个路由进程的顶点，以及每对路由邻接关系的有向边，Tiramisu 在这个图上运行特定域的 “tainting” 算法，来确定哪些路由进程可能知道给定前缀 p 的路由
- TPGs：除了邻接关系，还需要目的路由，如果第2层是 BGP，则需要第3层路由到邻接进程、以及物理连接
  - 每个设备和路由进程都创建一个顶点
  - 每条有向边对依赖关系进行建模：包括同一个路由器处于不同层次上的顶点，都需要用有向边来表示
  - 每条有向边会分配一个多属性标签，以便于对路由相关的过滤器和开销进行编码

#### 验证算法

已有的实践和学术研究将网络中的验证策略分为3类，验证进程在这个基础上进行。策略1：关心特定故障下的采用的实际路径：例如路径首选项；策略2：关心定量的路径指标：例如存在多少路径、路径长度边界；策略3：关心路径的存在性。

验证第一类策略时，需要对 Control Plane 的输出进行高保真建模，即精确转发路径的枚举/计算；验证最后一类策略时，对 Control Plane 进行低保真建模，即单个可能路径的证据。因此，Tiramisu 引入特定类别的验证算法，在确保精确度的前提下以最低的保真度运行。

- **TPVP**：对于验证策略1，Tiramisu 用 TPVP（扩展了 SPVP） 协议高效的解决了稳定路径问题。每个 TPG 节点使用出边的多维属性，使用简单的算术运算在多个可用路径中选择。在简单网络中，TPVP 退化成距离向量算法，在多项式时间内计算出故障路径，这种情况下和 ESMC 工具性能相当，比通用的 SMT 快。通常的网络中，TPVP 使用基于特定域的计算方法，与基于 SMT 的常规搜索策略快。并且它一次性模拟 Control Plane，而不是一次只探索一个状态，比 ESMC 快。
- **ILP**：对于策略2，Tiramisu 利用了通用的线性规划编码来计算通过 TPG 的网络流，而不是计算精确路径。ILPs 只考虑影响路径是否可以实现的协议属性，避免了路径计算。
- **TDFS**：Tiramisu 使用的新的多项式时间复杂度的图遍历算法（深度优先搜索），来检查策略3中路径的存在性。TDFS 只会对标准 DFS 进行恒定次数的调用，每次调用都是处理 TPG 中的一个子图（基于标签的带有一条路径的路由过滤器之间的交互进行建模），来控制是否可以实现。Tiramisu 利用了基于 Yen 的动态规划算法来计算 k 最短路径，直接计算在任何故障下显示的路径的首选项顺序。发展出的变体 TYEN 在 Yen 的执行上以有限的方式来调用 TPVP，减少了路径探索。TYEN 只能用于计算路径指标是单调的网络。

### Tiramisu 图抽象

RAG、TPG。TPG 基于 RAG，都基于网络配置和物理拓扑。

#### RAG

对路由邻接关系编码，Tiramisu 可以知道哪个路由进程可能知道特定 ip 的路由子网。

- **顶点**：每个顶点都表示一个路由进程或者设备（设备中包含 RAG 子网的静态路由）
- **边**：每条边表示一个路由邻接关系，如果两个 BGP 进程被配置为邻居，则是邻接的；如果两个 OSPF 进程被配置为同一第2层广播域中的路由接口上运行，则是邻接的；这些邻接关系用有向边来表示。用虚线来表示 iBGP 邻接关系。当同一个设备的一个进程向另一个进程分发路由时，这也构成一个邻接关系
- **Taints**：为了确定哪个路由进程可能知道特定的路由目的，Tiramisu 在 RAG 上运行 ”tainting“ 算法，所有与 RAG 关联的子网发起路由的顶点都会受到污染，污染通过边传播到其他的顶点（污染穿越一个 iBGP 边之后不能马上穿越另一个 iBGP 边）。这个算法假设所有的邻接关系都是活跃的，没有路由是过滤的。但是在真的网络中，如果一个邻接关系是活跃的，必须满足几个条件。

#### TPG

如果满足下列的依赖项，路由 R 上的进程 P 则可以将子网 S 的路由通告给路由 R' 上的相邻路由进程 P'

- P 从相邻进程中学习 S 的路由，或者 P 配置为 S 发起路由
- P 或 P' 均不过滤通告
- R 上的进程或 switch 知道 R' 的路由，或者 R 与 R' 在同子网/layer2 域内
- R 和 R' 有一个或多个物理连接

TPG 对这些依赖进行编码，Tiramisu 为网络中的每个 ip 子网构建一个 TPG

**顶点**：表示每个路由器上存在的路由信息库（RIB）和每个路由器/交换机上的（逻辑）流量入口/出口点，每个路由进程维护自己的 RIB。

**边**：表示流量可能采取的虚拟和物理的 ”跃点“。对以下的依赖关系建模：

- Layer1 & 2
- 依赖于 L2 的 OSPF
- BGP 与其依赖的 OSPF 路由
- 目的路由

**过滤器**：RAG 可能会过高的估计进程知道子网路由，因为没有考虑包过滤器。TPG 通过边缘修剪和边缘属性对过滤器建模。Tiramisu 使用边缘修剪对基于前缀或邻居的过滤器进行建模。会把过滤器相关联的出边/入边移除

**开销/偏好**：指标向量，根据边的不同，具体的指标可能会不同

### 策略类别1

#### TPVP（Tiramisu 路径向量协议）

SPVP 和 路由代数，没有考虑同一域内不同协议之间的依赖关系；Plankton 考虑了这个，但是在 RPVP 中用了集中处理的方式来解决这个问题

- 路由代数：为路由协议的路径开销计算和路径选择算法建模
- TPVP：扩展了 SPVP，使用了共享内存，而不是消息传递；它通过计算路径签名和使用对应于不同协议的路由代数操作选择路径，对多个协议进行串联建模

#### Tiramisu Yen 算法

### 策略类别2

#### Tiramisu Min-cut：

只考虑必要的属性，当使用包过滤（ACLs）时，ILP 不准确。

#### Tiramisu 最长路径

### 策略类别3

TDFS 多次调用 DFS 统计标签



### 验证效率

- 生成 TPG 所需时间：大型网络约等于 1ms
- 验证各种策略：WAYPT 使用了 TDFS，和 BLOCK 相同
  - BLOCK < 3ms
  - PREF 画的时间比 BLOCK 多
  - KFALL、BOUND 最慢，约等于 80ms
- 与其他工具比较
  - Minesweeper：PREF 加速不明显，BOUND 50X，使用 TDFS 的策略加速高达 600 ~ 800X；即使没有故障发生，Tiramisu 在所有的策略方面都比 Minesweeper 好；在验证 KFALL 和 BOUND 时，使用的变量比 Minesweeper 少 10 ~ 100X
  - Plankton：对于 iBGP 网络性能很差，对于大型网络，内存不足错误；小型网络，Tiramisu 比 Plankton 快 100 ~ 300X；在只有 OSPF 协议时，Plankton 可能快速尝试了故障连接，因此可能会快一点，但是不考虑这个，Tiramisu 在只有 OSPF 协议的网络上比 Plankton 快 2 ~ 50X
  - Batfish：快 70 ~ 100X
  - Bonsai：快 9X
- 规模大小：在大型网络上 < 0.12s
  - PREF 和 BOUND 相当，KFALL 时间明显较长，BLOCK 时间最短
- TYEN 加速：小型网络加速达 1.4X；大型网络加速达 3.8X

### 扩展和限制

优点：虽然只描述了 BGP 和 OSPF，同样可以用于其他的协议（RIP 和 EIGRP），也可以对 VRFs 建模

缺点：

- 没有对 advertisements 建模，因此无法确定是否存在可能违反策略的外部 advertisements；
- 只能穷尽地探索链路故障
- 必须提供具体的外部 advertisements 实例，才可以分析给定 advertisements 下的网络，是否可能违反任何策略
- 无法验证 Control Plane 是否等价
- 不是普遍的替代物
- 不能检查定量的 advertisements 
- 在 TPG 中对包过滤器的建模只考虑了基于 ip 的过滤，因此可能会误判可达性
- 没有考虑影响路由 advertisements 的包过滤器
- 无法对 iBGP 导致的路由偏转进行建模
- 无法对 iBGP 进程为从其 iBGP 邻居接收的路由分配首选项（lp）或基于标签的过滤器进行建模




