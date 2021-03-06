# CP与AP的取舍

上文我们讨论过ACID、基于2PC、TCC等方案的分布式事务、FLP Impossibility、拜占庭与非拜占庭下的领导者选择及其Paxos、Raft等实现算法，而这之中最核心的问题在于调和分布式环境的数据一致性与服务可用性。对此在2002年来自MIT的Seth Gilbert 和Nancy Lynch教授在[《Brewer's conjecture and the feasibility of consistent, available, partition-tolerant web services》](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf)论文中证明了Eric Brewer于早前提出的猜想并正式确立了CAP理论。于今天而言在众多的分布式系统架构基础理论中CAP可能为人所知，本节我们花些篇通俗地介绍下CAP及其相关概念。

计算机机专家Eric Brewer于2000年在ACM分布式计算原理专题讨论会（PODC）中提出的分布式系统设计要考虑三个核心要素：一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance），而这三个特性不可能同时满足。三个特性的说明如下：

* **一致性（Consistency）** 同一时刻同一请求的不同实例返回的结果相同，这要求数据具有强一致性(Strong Consistency)，需要特别说明的这里的一致性与ACID（见上文）的一致性不是同一个概念，不要混淆
* **可用性（Availability）** 所有读写请求在一定时间内得到正确的响应
* **分区容错性（Partition tolerance）** 在网络异常情况下，系统仍能正常运作

CAP认为分布式环境下网络的故障是常态，比如我们多机房部署下机房间就可能发生光缆被挖断、专线故障等网络分区情况（导致部分节点无法通信，原本一个大集群变成多个独立的小集群），也可能出现网络波动、丢包、节点宕机等，所以分布式系统设计要考虑的是在满足P的前提下选择C还是A。

抛开严谨的学术证明我们设想工作中的例子：我们要开发一个分布式缓存服务，只提供简单的读取与写入功能，服务支持多个节点做数据冗余及负载，请求由网关随机分发到其中一个节点，我们必须确保其中一个或几个节点故障时另一些节点仍然可以提供服务，在网络分区形成独立小集群时也可以提供服务，这就必须满足分区容错性（P），我们假设部署了两个服务节点，那么：

如果要保证一致性（C），即所有节点可查询到的数据随时随刻都是一致的（同步中的数据不可查询），就要求一个节点写入数据后必须再将数据写入到另一个节点后才能返回成功，这样当我们读取之前写入的数据时才能确保一致，但上文说明过网络异常在所难免，如果两个服务节点无法相互通讯时为保证一致性在数据写入发现无法同步到另一节点时就会返回错误进而牺牲了可用性（A）。

如果要保证可用性（A），即只要不是服务宕机所有请求都可得到正确的响应，那么在网络异常节点不能通讯的情况下要让数据没有同步到另一节点的请求也返回成功，这就必须牺牲一致性（C）导致在一段时间内（网络异常期间）两个服务节点所查询到的数据可能不同。

所以从中可以简单地发现一致性（C）与可用性（A）是不可能同时满足的。同FLP Impossibility 一样CAP理论也为我们做分布式服务架构指明了方向：分布式系统中我们只能选择CP（满足一致性牺牲可用性）或AP（满足可用性牺牲一致性）。

当我们选择CP，即满足一致性而牺牲可用性时意味着在网络异常出现多个节点孤岛时为了保证各个节点的数据一致系统会停止服务，反之选择AP，即满足可用性牺牲一致性时网络异常时系统仍可工作，但会出现各节点数据不致的情况。

在我们做微服务架构时需要知道CAP并做出架构设计或选型。比如注册中心常用的Eureka和Zookeepr实现，Eureka是AP的，Zookeeper是CP的，Spring Cloud之所以推荐Eureka是因为它认为注册中心的场景允许出现短暂的数据不一致情况，可用性要高于强一致性，再比如数据库HBase与Cassandra，两者同为NoSQL数据，部分需求两者都可满足，但我们要考虑允不允许出现数据不一致，HBase是强一致性的，Cassandra则是弱一致性的，但换来了更好的可用性。

上面出现了“强一致性”与“弱一致性”两个概念，这其实是对一致性的延展，大量的工程实践的经验表明可用性很重要，一致性也很重要，但可以容许一定的时差，即只要保证在一定时间内达到一致即可，这也就是所谓的最终一致性。要实现强一致性的成本很高，尤其是存在很多数据副本的情况下，区块链的PoW及其衍生算法就是典型的代表，它的共识机制是概率强一致性（Probabilistic Strong Consistency），要求等待大多数节点都接受了这笔交易再真正接受它，但是带来的问题是交易的确认严重滞后。基于此出现了[Base理论](https://dl.acm.org/citation.cfm?id=1394128)。

Base是基本可用（Basically Available）、软状态（Soft State）和最终一致性（Eventual consistency）三个短语的简写，是对CAP的扩展。正如上面所说，Base方案允许系统在一段时间内存在数据不一致性，存在软状态，但在规定的时间后数据会最终一致性。这样做的好处在于满足可用性的同时也在一定程度上符合一致性的要求。回顾我们上面的缓存例子，如果我们缓存的是商品的列表用于检索（考虑淘宝商品搜索结果），那么多半业务场景下是允许存在一定的同步时间即相近时间的两次相同请求到不同服务节点所返回的结果可以不同，这就符合Base方案，当然如果是实现类似商品库存的缓存那多半业务场景是要保证强一致性的。

分布式系统发展至今产生了很多重要的理论与定理，上文只讨论了其中比较知名的几个，围绕一致性问题就还有诸如PACELC理论 、一致性模型 （Consistency model ）等更为基础的或小众的研究结果，前人基于这些学术研究构建了丰富多样的分布式服务，而微服务正是分布式服务的一种形态。
