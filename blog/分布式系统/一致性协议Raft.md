### 和Paxos的关系
Paxos是一种强一致性协议，在分布式系统容错处理上有广泛使用。Paxos协议使用的前提是没有拜占庭将军问题，即节点可以各种不稳定但不会故意欺骗。

>- Agents operate at arbitrary speed, may fail by stopping, and may
restart. Since all agents may fail after a value is chosen and then
restart, a solution is impossible unless some information can be remembered by an agent that has failed and restarted.
>- Messages can take arbitrarily long to be delivered, can be duplicated,
and can be lost, but they are not corrupted.

Paxos的论文在这里[https://lamport.azurewebsites.net/pubs/paxos-simple.pdf](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf "https://lamport.azurewebsites.net/pubs/paxos-simple.pdf")

Paxos将角色分为三类：提议者 (Proposer)、决策者 (Acceptor)、最终决策学习者 (Learner)。每个节点可以同时是多种角色（在不同的提案里）。
决策过程分为两个阶段：Prepare和Accept。
下面一个提案的协商过程。

![basic-paxos.jpeg](https://ping666.com/wp-content/uploads/2024/09/basic-paxos.jpeg "basic-paxos.jpeg")

这个协商过程有两个缺点：

1. 过程复杂难以理解(允许并发提案)，状态机很容易进入死循环。
1. 每个提案都要经过2次投票协商，效率有点低。

其中的一种优化方案就是Raft:

1. Leader选举策略
   选举出一个Leader，由这个Leader一直提案。直到他出现故障（或者到达一定的任期）后重新选举。
2. 顺序提案
   选举出的Leader，按顺序提案。即使同一时期有多个提案共存，但是整体上是FIFO的。

基于以上两个假设，Raft的实现比Paxos更容易理解，性能更好，而且满足大多数场景。

### Raft简介
Raft的白皮书可以看这里[https://raft.github.io/raft.pdf](https://raft.github.io/raft.pdf "https://raft.github.io/raft.pdf")

>Raft implements consensus by first electing a distinguished leader, then giving the leader complete responsibility for managing the replicated log. The leader accepts log entries from clients, replicates them on other servers, and tells servers when it is safe to apply log entries to their state machines. Having a leader simplifies the management of the replicated log. A leader can fail or become disconnected from the other servers, in which case a new leader is elected.

上面这一段高度概括了Raft的流程：

1. 选举Leader
1. Leader负责从客户端接收日志、分发日志给其他节点、告诉其他节点应用日志
1. 如果Leader故障或者工作一段时间（每个任期term）后，就重新选举

### 重要概念

- 选举Leader
  Followers在election timeout时间内没有收到Leader的心跳，就term+1进入candidate状态，同时向其他节点发起投票。等待应答有三种情况：
  1. 得票多者成为新任期的Leader
  2. 有其节点已经成为Leader，重新进入Follower
  3. 一段时间内还没有胜出者，重新发起投票
![raft-leader-election.png](https://ping666.com/wp-content/uploads/2024/09/raft-leader-election.png "raft-leader-election.png")

- 日志复制
>Each log contains the same commands in the same order, so each state machine processes the same sequence of commands. Since the state machines are deterministic, each computes the same state and the same sequence of outputs.

  **同样的输入命令 + 同样的执行顺序 + 同样的执行程序 = 同样的结果**
程序是一致的，所以Leader只需要保证前两项就可以了。Leader采用和关系型数据库类似的两段提交（P2C），只是提交前加了一步：收集多数节点的投票。
![raft-log.png](https://ping666.com/wp-content/uploads/2024/09/raft-log.png "raft-log.png")
上面是正常的Leader到Followers的共识复制。还有一种场景是新加入的节点从Leader批量同步差异数据。

- 数据安全
  当Leader失效后，各个节点都能发起candidate，但只有term最大的那批节点中的一个会成功（也就是最后一次正常的状态，也即数据最新的那批节点）。

### Raft应用
很多开源软件使用了Raft协议，特别是数据存储这块，包括Tidb、RocketMQ、consul、etcd等。

RocketMQ的Raft实现Dledger，项目地址在[https://github.com/openmessaging/dledger.git](https://github.com/openmessaging/dledger.git "https://github.com/openmessaging/dledger.git")
