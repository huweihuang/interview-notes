# cgroup和namespace

## namespace（访问隔离）

Namespaces机制提供一种访问隔离方案。PID,IPC,Network等系统资源不再是全局性的，而是属于某个特定的Namespace。每个namespace下的资源对于其他namespace下的资源都是透明，不可见的。

## cgroup（资源控制）

CGroups可以限制、记录、隔离进程组所使用的物理资源（包括：CPU、memory、IO等），为容器实现虚拟化提供了基本保证。CGroups本质是内核附加在程序上的一系列钩子（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。

# Raft算法

该系统中有三种角色，follower,candidate,leader。起初所有的角色都是follower，当他没有收到leader的消息，就将自己设置为candidate，同时向其他节点发起投票，如果大部分节点同意他当leader，则他则被选为leader。当客户端写一个值通过leader，leader将值同步给其他节点，当其他的大部分节点将值写入，并返回给leader，leader提交了再返回给客户端，保证一致性。

## 选举

有两个timeout来控制选举，第一个是`election timeout`，该时间是节点从follower到成为candidate的时间，该时间是150到300毫秒之间的随机值。另一个是`heartbeat timeout`。

- 当某个节点经历完`election timeout`成为candidate后，开启新的一个选举周期，他向其他节点发起投票请求（Request Vote），如果接收到消息的节点在该周期内还没投过票则给这个candidate投票，然后节点重置他的election timeout。
- 当该candidate获得大部分的选票，则可以当选为leader。
- leader就开始发送`append entries`给其他follower节点，这个消息会在内部指定的`heartbeat timeout`时间内发出，follower收到该信息则响应给leader。
- 这个选举周期会继续，直到某个follower没有收到心跳，并成为candidate。
- 如果某个选举周期内，有两个candidate同时获得相同多的选票，则会等待一个新的周期重新选举。

## 日志同步

当选举过程结束，选出了leader，则leader需要把所有的变更同步的系统中的其他节点，该同步也是通过发送`Append Entries`的消息的方式。

- 首先一个客户端发送一个更新给leader，这个更新会添加到leader的日志中。
- 然后leader会在给follower的下次心跳探测中发送该更新。
- 一旦大多数follower收到这个更新并返回给leader，leader提交这个更新，然后返回给客户端。

## 发生网络分区

- 当发生网络分区的时候，在不同分区的节点接收不到leader的心跳，则会开启一轮选举，形成不同leader的多个分区集群。
- 当客户端给不同leader的发送更新消息时，不同分区集群中的节点个数小于原先集群的一半时，更新不会被提交，而节点个数大于集群数一半时，更新会被提交。
- 当网络分区恢复后，被提交的更新会同步到其他的节点上，其他节点未提交的日志会被回滚并匹配新leader的日志，保证全局的数据是一致的。

# CAP理论

一致性，可用性，分区容忍性，三者不能同时全部满足，只能满足其中两个。

c：一致性，表示写入一个值后的读操作必须返回该值，即前后一致的值

a: 可用性，只要收到客户端的请求，服务端必须给出返回值。

p: 分区容忍性，可以容忍不同分区节点间的丢包或延迟问题

一般情况下，网络分区一定会存在，因此不得不容忍不同分区的丢包问题。所以一般在一致性和可用性之间做权衡。

证明：反证法

假设存在一个系统满足三个特性，该系统有G1,G2两个server，G1和G2因为满足分区容忍性，即存在丢包，当C给G1写一个数据，G2与G1之间因为丢包，G2上对应的值没有被更新，那么C向G2读取该值的时候，读取到了旧的值，这导致C在写后读取到的值不一致，因此不满足一致性。因此满足三个特性的系统不存在。

## CP without A

一般的分布式系统，redis zk都大部分保留一致性，牺牲可用性

## AP wihtout C

10306买票，前后票不一致问题，保证可用性

