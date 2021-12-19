很多人喜欢把 Paxos 说成是一致性协议，但是一致性这个词其实会给大家误导，会把这个一致性和 ACID
的一致性（Consistency）联想在一起，Paxos 的“一致性”实际上是
consensus，中文翻译成“共识”可能会更加准确一点，这个“共识”更强调的是多副本状态的一致性。

**本篇面试内容划重点：Paxos、Raft 的主要内容。**

### Paxos 协议

Paxos 是最基础的分布式共识算法，用于解决分布式系统中多副本一致性问题，它可以说是这一领域的奠基石，后面基本所有的相关算法都是基于它做的改造。

#### **角色说明**

  * **Client 产生议题者** ：即客户端，发起请求产生议题的角色，实际不参与选举过程。
  * **Proposer 提议者** ：Proposer 可以有多个，负责接受 client 的请求，并向 acceptor 发起提议。提交分为两个阶段，具体流程下面有详解。
  * **Acceptor 决策者** ：Acceptor 负责接受 Proposer 的议案，但是议案必须获得超过半数（N/2+1）的 Acceptor 确认后才能通过。
  * **Learner 最终决策学习者** ：Learner 不参与选举，但是会去获取决策结果，在 Paxos 中，Learner 主要参与相关的状态机同步流程。

![image.png](https://images.gitbook.cn/095c9e60-f5d7-11ea-a625-2d171281165b)

#### **协议的执行流程**

**准备阶段（第一阶段）**

1\. Proposer 选择一个提议编号 n，向所有的 Acceptor 广播 Prepare(n) 请求。

2\. Acceptor 接收到 Prepare(n) 请求：

  * 若此时 Acceptor 没有接收过请求，则接受该提议，并返回为 null 的 value 值（说明所有的 Proposer 请求都还在准备阶段）。
  * 若 Acceptor 此前接收到的请求的 n 值大于当前的 n 值，则拒绝接受该提议，并返回已经接受的提议中最大的 n 对应的 value 值（说明已经有 Proposer 完成了选举阶段）。
  * 反之，此前接收到的请求的 n 值小于当前的 n 值，则接受该提议。

**选举阶段（第二阶段）**

1\. 准备阶段过了一段时间后，Proposer 会收集到一些 Prepare 回复。

  * 如果有一半以上的 Acceptor 在第一阶段回复了接受的信息，且所有回复的 value 都为 null 时，则 Porposer 发出 accept 请求，并带上自己的 value。
  * 如果有一半以上的 Acceptor 在第一阶段回复了接受的信息，但回复的消息中存在 value 不为空的情况，则 Porposer 发出 accept 请求，但不是带上自己的 value ，而是回复消息中的 value（跑票了）。
  * 如果回复的消息没有达到一半以上，则生成更大的提议编号 n，重新回到第一阶段。

2\. Accpetor 收到第二阶段的 Accpet 请求后，判断：

  * 如果这次收到的 N 与第一阶段的 N 相同，则提交成功，记录下 N 和 value 值；
  * 如果这次收到的 N 小于第一阶段的 N，则提交失败。

3\. 经过一段时间后，Proposer 会收集到一些 Accept 回复提交成功的情况，比如：

  * 当 Proposer 收到一半以上的 Accpetor 回复提交成功的消息，则表示提交 value 成功，广播告诉所有 Proposer、Learner 提交成功的 value；
  * 若没有收到一半以上的回复，则生成更大的提议编号 n，重新回到第一阶段。

**为什么要用“半数以上通过”这个办法来决策？**

一个集合不可能同时存在两个半数以上的子集，过半的思想是为了保证提交的 value 在同一时刻在分布式系统中是唯一的一致的。

当 Proposer 有很多个的时候，会有什么问题？

很难有一个 proposer 收到半数以上的回复，进而不断地执行第一阶段的协议，决策收敛速度慢，很久都不能做出一个决策，这其实就是 Basic-Paxos
存在的问题，Mulit-Paxos 在这方面做了优化，只允许一个 proposer 提议，可以加速算法收敛。

### Raft 协议

上面我们说的是 Basic-Paxos，Mulit-Paxos 基于 Basic-Paxos 做了优化，在 Paxos 集群中利用 Paxos
协议选举出唯一的 Leader，在 Leader 有效期内所有的议案都只能由 Leader 发起。 下面要说的 Raft 就是类似 multi-Paxos
的一种协议（Zab 也是）。

#### **角色说明**

  * Leader：处理所有客户端交互，日志复制等，同一时刻的 Leader 是唯一的。
  * Follower：从 Leader 被动接受更新请求，然后写入本地日志文件
  * Candidate：候选者，参与 Leader 的选举，在 Follower 一段时间内没有收到 Leader 的心跳，启动选主过程后，Follower 会变成 Candidate 状态，直到选主结束。

![image.png](https://images.gitbook.cn/9c20f200-f5d7-11ea-a9e7-47fb41a2df40)

#### **协议的执行流程**

**选举阶段**

  1. 节点启动后是处于 Follower 状态，如果一段时间没有收到来自 Leader 的心跳消息或者其它 Candidate 的请求投票消息，会转为 Candidate，发起新一轮选举过程。（term+1）
  2. 节点变成 Candidate 后，会向其它节点发送投票信息，如果一段时间后当前节点的角色状态没有发生变化，则会再次发起一轮新的选举。
  3. 节点处于 Candidate 状态时，如果收到来自 Leader 的心跳消息，就会立即变身为 Follower。如果发出去的投票请求得到了半数节点的成功回应，就会立即变身为 Leader，并周期性地向其它节点广播心跳消息。
  4. 日志都会有一个有序的编号 LogIndex 和它被创建时的任期号 term （每次选举 term+1）， LogIndex 和 term 就是 Leader 选举的依据，凡是 Candidate 收到的选票中 LogIndex 和 term 没有自身大的选票都拒绝，如果比自身大则接受。
  5. 有半数以上的 Candidate 确认了同一个 Candidate 成为 leader 则选举结束。

**日志同步阶段**

在 Leader 发生切换的时候，新 Leader 的日志和 Follower 的日志可能会存在不一致的场景。因为 Leader
的数据是只能增加不能删除的，所以 Follower 就需要对日志进行截断处理，再从截断的位置重新同步日志，过程如下。

  1. Leader 当选后立即向所有节点广播消息，携带最新一条日志的信息。Follower 收到消息后和自己的日志进行比对，如果 leader 的日志和自己的不匹配则返回失败信息。
  2. Leader 收到失败消息后，尝试将最新的两条日志发送给对应的 Follower。如果 Follower 发现日志还是不匹配则再返回失败，如此循环往复直到成功找到 Leader 与 Follower 一致的日志 index，然后向后逐条覆盖 Follower 在该位置之后日志，如此来保证 Follower 和 Leader 的数据一致。
  3. 每一个 Follower 节点都会重复上面的步骤与 Leader 同步数据直到系统的数据一致。

Zab 协议在 Zookeeper 的章节已经详细说过了，这里就不聊了，总结一下 Raft 和 Zab 的区别：

**相同：**

  * 使用 Quorum 来确定系统的一致性；
  * 用 heartbeat 来检测存活，超时则触发选举流程；
  * 采用相似的 **最新日志标识** 和 **选举轮次** 来作为选举 Leader 的依据。

**不同：**

  * Raft 协议数据单向从 Leader 到 Follower 同步数据，Zab 中还有一个将自己的 log 更新为 Quorum 中最新的 log 的步骤，然后才向其他的节点同步数据。

**Leader 崩溃会触发选举，Follower 和 Candidate 崩溃呢？**

单个 Follower 或 Candidate 崩溃对集群的一致性不会造成太大的影响，Raft 会不断地重试连接，直到服务重启再同步这段时间遗漏的数据。

**异常场景下，如何保证 Client 端的指令仅执行一次？**

Leader 宕机、网络超时等原因都会造成 Client 端未收到响应从而发起重试引起指令重复执行等问题。为了解决这个问题，Client
端的每一条指令都会赋予一个事务 id。状态机会跟踪每条指令最新的 id 和相应的响应。如果接收到一条指令，它的 id
已经被执行了，那么就立即返回结果，而不重新执行指令。

