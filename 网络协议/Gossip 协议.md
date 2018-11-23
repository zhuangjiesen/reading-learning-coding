# Gossip 协议

一个分布式集群中的节点信息传播的协议(redis集群)

首先这是个英语单词 gossip 

### 百度翻译：
```
gossip	英[ˈgɒsɪp]
美[ˈgɑ:sɪp]
n.	流言蜚语，谣言; 爱讲闲话的人; 谈话，闲话; 关系亲密的伙伴;
vi.	传播流言，说长道短;
[例句]He spent the first hour talking gossip
他头一个小时都在闲聊。
[其他]	第三人称单数：gossips 复数：gossips 现在分词：gossiping 过去式：gossiped
```

### 维基百科文档：
https://en.wikipedia.org/wiki/Gossip_protocol

### 翻译

gossip 协议是一种电脑(服务)之间通信的程序或者过程，基于社交网络的信息留言传播和传染病的扩散方法；
gossip 是一种通信协议，现代分布式系统经常使用它来解决使用其他方法难以解决的问题，或者因为网络条件下不方便的架构，或者是gossip更有效，更能达到可用性


### Gossip communication
Gossip 协议的概念就类似于办公司里的流言蜚语传播，每个小时员工都会聚集在冷却机边上，每个员工随机地与另外一个凑成一对，分享最近的流言(gossip)，在一天的开始Alice 最先开始一个新的谣言，他对Bob 评论的说到，Charlie 染了他的胡子，在接下去的回忆，Bob告诉 Dave ，同时Alice 又告诉了Eve 。经过每一次冷却器上的小聚，听说这个谣言的人员数量已经大概翻倍了（虽然并没有同时传播给同一个人，可能Alice 尝试要告诉 Frank ，却发现他已经从Dave 那里知道了）。计算机系统典型地实现这样给定频率条件下以随机形式地同类选择的协议，每台机器都随机地选择另外一台机器并分享新的谣言

Gossip 的能力体现在健壮的传播消息，即使Dave 无法理解 Bob ，他也将会很快的从另外的人那里得到最新的消息

gossip 满足一下的场景：

```
· 协议的核心包括周期性，两个两个地，进程间的相互作用
· 这些信息交换的相互作用是有限制的
· 一发生通信，至少有一个节点的状态会影响另一个节点的状态
· 并不是(被认为)可靠的通信
· 通信频率很低与特定的消息相比是低延迟的，所以它的消耗几乎是忽略不计(微乎其微)的
· 节点相互选择中存在一定的随机性，节点可能在所有节点中选择也可能在相邻节点(数据集比较小)中选择
· 由于重复(同一个节点可能接受多个其他节点的消息)，所以消息可能会出现潜在的冗余
```

### Gossip protocol types
三种流行的gossip 应用类型

```
· Dissemination protocols
· Anti-entropy protocols 
· Protocols that compute aggregates
```


```

Dissemination protocols (or rumor-mongering protocols). These use gossip to spread information; they basically work by flooding agents in the network, but in a manner that produces bounded worst-case loads:
Event dissemination protocols use gossip to carry out multicasts. They report events, but the gossip occurs periodically and events don’t actually trigger the gossip. One concern here is the potentially high latency from when the event occurs until it is delivered.
Background data dissemination protocols continuously gossip about information associated with the participating nodes. Typically, propagation latency isn’t a concern, perhaps because the information in question changes slowly or there is no significant penalty for acting upon slightly stale data.
Anti-entropy protocols for repairing replicated data, which operate by comparing replicas and reconciling differences.
Protocols that compute aggregates. These compute a network-wide aggregate by sampling information at the nodes in the network and combining the values to arrive at a system-wide value – the largest value for some measurement nodes are making, smallest, etc. The key requirement is that the aggregate must be computable by fixed-size pairwise information exchanges; these typically terminate after a number of rounds of information exchange logarithmic in the system size, by which time an all-to-all information flow pattern will have been established. As a side effect of aggregation, it is possible to solve other kinds of problems using gossip; for example, there are gossip protocols that can arrange the nodes in a gossip overlay into a list sorted by node-id (or some other attribute) in logarithmic time using aggregation-style exchanges of information. Similarly, there are gossip algorithms that arrange nodes into a tree and compute aggregates such as "sum" or "count" by gossiping in a pattern biased to match the tree structure.
Many protocols that predate the earliest use of the term "gossip" fall within this rather inclusive definition. For example, Internet routing protocols often use gossip-like information exchanges. A gossip substrate can be used to implement a standard routed network: nodes "gossip" about traditional point-to-point messages, effectively pushing traffic through the gossip layer. Bandwidth permitting, this implies that a gossip system can potentially support any classic protocol or implement any classical distributed service. However, such a broadly inclusive interpretation is rarely intended. More typically gossip protocols are those that specifically run in a regular, periodic, relatively lazy, symmetric and decentralized manner; the high degree of symmetry among nodes is particularly characteristic. Thus, while one could run a 2-phase commit protocol over a gossip substrate, doing so would be at odds with the spirit, if not the wording, of the definition.

Frequently, the most useful gossip protocols turn out to be those with exponentially rapid convergence towards a state that "emerges" with probability 1.0. A classic distributed computing problem, for example, involves building a tree whose inner nodes are the nodes in a network and whose edges represent links between computers (for routing, as a dissemination overlay, etc.). Not all tree-building protocols are gossip protocols (for example, spanning tree constructions in which a leader initiates a flood), but gossip offers a decentralized solution that is useful in many situations.

The term convergently consistent is sometimes used to describe protocols that achieve exponentially rapid spread of information. For this purpose, a protocol must propagate any new information to all nodes that will be affected by the information within time logarithmic in the size of the system (the "mixing time" must be logarithmic in system size).
```



