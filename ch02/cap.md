# 分布式系统的CAP定理
## 简介
*CAP定理*: 指的是在一个分布式系统中，一致性(`Consistency`)、可用性
(`Availability`)、分区容错(`Partition tolerance`)这三个要素最多只能同时实现两点，
不可能三者兼顾。下面是一个更直观的图，图片来源[Distributed Systems for fun and profit](http://book.mixu.net/distsys/abstractions.html)
[图2-1](CAP.png)

1. *一致性(`C`)*: 这里的一致性不等价于数据库*ACID*中的一致性，也不等价于*Oatmeal*一致
   性，它是针对分布式系统最基本的提升可用性的方式分区复制(副本)。因此这里的一致
   性就是分区副本之间的一致性。此外，这里的一致性指的是强一致性，即所有副本保持
   一致。
1. *可用性(`A`)*: 在部分节点失败的情况下，系统仍然可用，分区和复制是分布式系统中提升可用性的基本
   方式。
1. *分区容错(`P`)*: 这里的分区特指网络的分区，与数据的分区不同。来自[CAP定理的含
   义](http://www.ruanyifeng.com/blog/2018/07/cap.html) 的解释比较直观：大多数分
   布式系统都分布在多个子网络，每个子网络就叫做一个区(`Partition`) 区间通信可能
   失败，从而造成网络分区。 而在系统设计的时候必须考虑这种情况。

## `C`、`A`、`P` 之间的关系
理解 CAP 理论的最简单方式是想象两个节点分处分区两侧。允许至少一个节点更新状态会
导致数据不一致，即丧失了 C 性质。如果为了保证数据一致性，将分区一侧的节点设置为
不可用，那么又丧失了 A 性质。除非两个节点可以互相通信，才能既保证 C 又保证 A，这
又会导致丧失 P 性质。来自[CAP 理论十二年回顾："规则"变了](https://www.infoq.cn/article/cap-twelve-years-later-how-the-rules-have-changed/)

1. `C` vs `A`: 一致性和可用性是冲突的，可用性要求容忍部分节点失败，这种情况下继
   续接受写入操作，那么失败节点上的副本无法和接受写入的副本保持一致，降低了一致
   性。
1. `C` vs `P`: 一致性和分区容错是冲突的，如果要分区容错，必然使得其中一个分区失
   效，这就意味着所有副本不可能达成一致，降低了一致性。
1. `A` vs `P`: 可用性和分区容错是冲突的，如果要分区容错，那么某一个分区将不能提
   供服务(否则会引入分歧，降低一致性)。那么这会降低系统可用性，因为能够容忍的失
   败的节点更少了，降低了可用性。
   

1. `CA`系统: 不区分节点失败和网络失败，为了防止多副本之间的分歧，必须停止接受写
   入操作。因为不知道是节点失败还是网络失败，保险的做法就是停止接受写。
1. `CP`系统: 再发生网络分区的时候，只能保持大多数节点的分区继续工作，少部分节点
   的分区不可用。

## `CAP`权衡
一个全球分布式系统中（全球广域网分布式环境下），网络分区是一个自然的事实(相关参
考见[ Fallacies of Distributed Computing
Explained](http://www.rgoarchitects.com/Files/fallacies.pdf))。 因此，`C`、`A`、
`P`三者并不对 等:`P`是基础，`CA`之间权衡。但是这并不意味着没有`AP`系统，例如分布
式缓存系统.

然而如果网络分区很少发生(例如Google的Spanner)，就没有必要牺牲一致性和可用性。其
次，一致性有很多级别，可用性显然在0%到100%之间变化，我们可以在三种特性程度上进行
权衡。

## 参考文献
+ [CAP原则](https://baike.baidu.com/item/CAP%E5%8E%9F%E5%88%99/5712863?fr=aladdin)
+ [CAP理论与分布式系统设计](https://mp.weixin.qq.com/s/gV7DqSgSkz_X56p2X_x_cQ)
+ [CAP理论十二年回顾](https://www.infoq.cn/article/cap-twelve-years-later-how-the-rules-have-changed/)
+ [Distributed systems for fun and profit](http://book.mixu.net/distsys/abstractions.html)
+ [CAP定理的含义](http://www.ruanyifeng.com/blog/2018/07/cap.html)
