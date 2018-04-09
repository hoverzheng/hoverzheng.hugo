---
title: "Raft算法详解2--Raft算法概要"
date: 2018-03-27T06:55:36+08:00
draft: false
categories: "BlockChain"
tags: ["Raft", "共识算法", "blockchain"]
---

## Raft算法介绍
Raft是一个共识算法，用来管理复制日志(replicated log)，关于复制日志后面有讲述。本文以简明形式总结了该算法，还列出了该算法的关键属性。本文是对Raft算法的一个概述，还会有更加详细的章节专门介绍该算法的细节。

Raft实现了共识算法，它首先选举一个唯一的leader，并赋予该leader完全的管理复制志的职责。该leader从客户端接收日志记录，将它们复制到其他服务节点上，并告诉服务节点何时可以安全地将日志记录应用到其状态机(state machine)。有了leader后，就简化了复制日志的管理。例如：leader可以决定把新的记录放到日志中，而不询问其他服务节点，数据流以简单的形式从leader流向其他服务节点。leader可能会终止或与其他服务节点断开连接，此时会选举出新的leader。

在有leader的前提下，Raft把共识问题分解成独立的子问题，这些问题将在下面的章节进行讨论：

* 选主(Leader election)

&emsp;当现有的leader终止时，必须要选出一个新的leader。

* 日志复制(Log replication)

&emsp;leader必须接受客户端的日志记录，把它复制到整个集群，并强制使其他的日志和它自己的保持一致。

* 安全性(Safety)

&emsp;Raft关键的安全属性是状态机的安全属性，如下面的章节所示。如果任何服务节点已经在它的状态机上使用了一个特定的日志记录，那么其他服务节点不能在相同的日志索引上使用不同的指令。


### Raft算法的特性

Raft保证这些属性的每一个在任何时候都是true。后面会有文章对这些属性进行详细讨论。

* 选举安全(Election Safety)

&emsp;在给定的周期内，最多只有一个leader被选举出来。

* leader只追加日志(Leader Append-Only)

&emsp;leader从来都不会覆盖或删除它的日志记录。它仅追加新的日志记录。

* 日志匹配(Log Matching)

&emsp;如果两个日志包含相同索引和时间周期的记录，那么日志在给定索引下的所有记录都是相同的。

* Leader的完整性(Leader Completeness)

&emsp;如果日志记录是在给定的周期内提交的，那么该记录将出现在所有具有更高编号周期的leader的日志中。

* 状态机的安全性(State Machine Safety)

&emsp;如果服务节点在其状态机的给定索引处应用了日志记录，则其他服务节点将能使用具有相同索引号的不同的日志记录。

## Raft算法概要
下面是Raft共识算法的概括(不包括节点权限变更和日志压缩)。服务节点的行为被描述为独立和重复触发的一组规则。 

### 状态(State)
#### 1. 所有服务节点上持久化的状态
这些状态会在响应RPC之前进行更新，并持久化在服务节点。

* currentTerm

&emsp;服务节点已经观察到的最新的周期(初次启动时初始化为0，后面不断增加)。

* votedFor

&emsp;candidateId在当前周期内收到的投票（如果没有，则为null）

* log[] 

日志记录;每个记录包含状态机的命令，以及leader接收到日志实体时的周期(第一个索引为1)

#### 2. 所有服务节点上的易失性状态

* commitIndex

已知提交的最新日志实体的索引(初始化为0，单调递增)

* lastApplied

应用于状态机的最高日志实体的索引(初始化为0，单调递增)

#### 3. leader上的临时状态
这些状态会在选举后被重新初始化。

* nextIndex[] 

对于每个服务节点，发送到该服务节点的下一个日志条目的索引(初始化为leader的最后日志索引+1)

* matchIndex[] 

对于每个服务节点，已知在服务节点上复制的最高日志实体的索引(初始化为0，单调递增)

### 追加日志记录请求 (AppendEntries RPC)
领导者调用以复制日志条目，也用作心跳。

#### 1.参数(Arguments)
* term

leader的时间周期

* leaderId

follower可以重定向客户端

* prevLogIndex

日志实体在新实体之前的索引值

* prevLogTerm

prevLogIndex周期的日志实体

* entries[]

需要存储的日志记录。(若是心跳消息则为空；可以发送多个用于提高效率)

* leaderCommit

leader的commitIndex

#### 2.结果(Results)
* term

当前周期，用于leader的自我更新

* success

如果follower包含与prevLogIndex和prevLogTerm匹配的记录，则返回true

#### 3.接收者的实现(Receiver implementation)
(1) 若term < currentTerm 则返回false

(2) 如果日志中不包含prevLogIndex中与prevLogTerm相匹配的记录，则回复false

(3) 如果现有记录与新记录(相同索引但时间周期不同)冲突，删除现有记录和其后的所有记录。

(4) 添加日志记录中尚未包含的新记录

(5) 如果leaderCommit> commitIndex，则设置commitIndex = min(leaderCommit，最后一个新日志记录的索引)

### 投票请求(RequestVote RPC)

#### 参数(Arguments)
* term

candidate 的时间周期

* candidateId

candidate请求投票

* lastLogIndex

candidate最后一个日志记录的索引

* lastLogTerm

candidate最后一个日志记录的时间周期

#### 结果(Results)
* term

当前周期，用于candidate更新它自己

* voteGranted

若值为true，表示candidate收到了投票

#### 接收者的实现(Receiver implementation)
(1) 若term < currentTerm返回false

(2) 如果votedFor为null或candidateId，并且candidate的日志至少与接收者的日志一样最新，则授予投票

### 服务节点的规则
#### All Servers
* 如果commitIndex> lastApplied：增加lastApplied，则将log[lastApplied]应用于状态机

* 如果RPC请求或响应包含T> currentTerm：set currentTerm = T，则转换为follower

#### Followers
* 回应candidate和leader的RPC请求

* 如果超时没有收到当前leader的AppendEntries RPC请求或给candidate的投票请求：则转换为candidate状态

#### Candidates
* 转换为Candidates后，开始选举
   * 递增currentTerm
   * 为自己投票
   * 重置选举计时器
   * 发送RequestVote RPC到所有其他服务节点

* 如果从大多数服务器收到投票则：变成leader

* 如果从新的leader收到AppendEntries RPC请求则：变成follower

* 如果选举超时则：开始新的选举

#### Leaders
* 一旦选举：发送初始emptyAppendEntries RPC（心跳）到每个服务器; 在空闲期间重复以防止选举超时

* 如果从客户端接收到的命令：将条目附加到本地日志中，则在应用到状态机

* 如果追随者的最后日志索引≥nextIndex：将appendEntries RPC发送到从nextIndex开始的日志条目
   * 如果成功：为follower更新nextIndex和matchIndex   
   * If AppendEntries由于日志不一致而失败：递减nextIndex并重试

* 如果存在N，使得N> commitIndex，则大部分matchIndex [i]≥N，并且log [N] .term == currentTerm：set commitIndex = N。 


## 参考资料
* [raft.github.io](https://raft.github.io/)

