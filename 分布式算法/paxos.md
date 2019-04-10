# 基于状态机复制的共识算法:Paxos    学习笔记

## paxos简介


## Basic-Paxos

### 角色

- Client：请求发起者，系统外部角色
- Proposer：接收Client请求向集群提出提议
- Acceptor(Voter)：提议投票和接收者，只有形成法定人数（Quorum，一般为majortiy多数派）时，提议才会最终接受。
- Learner：提议记录，backup，不参与投票

### 基本成功流程
            Client   Proposer      Acceptor     Learner
    时    |         |          |  |  |       |  | --- First Request ---
          X-------->|          |  |  |       |  |  Request
    间    |         X--------->|->|->|       |  |  Prepare(N)
          |         |<---------X--X--X       |  |  Promise(N,I,{Va,Vb,Vc})
    线    |         X--------->|->|->|       |  |  Accept!(N,I,V)
          |         |<---------X--X--X------>|->|  Accepted(N,I,V)
    ↓     |<---------------------------------X--X  Response
          |         |          |  |  |       |  |

### 流程简述

参考上图自上而下的流程：
1. Request：Client发送请求
2. Prepare：第一轮RPC开始，Proposer向Acceptor发送编号N的提议，并且N大于之前的任何编号
3. Promise：Acceptor根据Proposer的编号N是否是当前最大的编号返回是否处理这个提议，N检测成功则继续，否则结束
4. Accept：第二轮RPC开始，Proposer发送提议内容给Acceptor
5. Accepted：Acceptor通过提议返回给Proposer并且Learner记录
6. Response: Learner返回执行结果给Client


### 潜在问题
- 难实现
- 2轮RPC效率低
- paxos存在活锁（dueling）的可能，即当Proposer第N个提议没有进行到Accept时，发来了N+1个提议，Acceptor抛弃了提议N，去处理提议N+1，提议N被丢弃后重新发起提议这时前面的提议也没有处理完，如此反复系统就形成了活锁。解决方案是在发生冲突时，给冲突的提议添加一个random的等待时间。

## Multi-Paxos

### 角色

- Client：请求发起者，系统外部角色
- Proposer：接收Client请求向集群提出提议（和Basic-Paxos区别在Proposer唯一）
- Acceptor(Voter)：提议投票和接收者，只有形成法定人数（Quorum，一般为majortiy多数派）时，提议才会最终接受。
- Learner：提议记录，backup，不参与投票

### 基本成功流程

    第一轮确定唯一的Proposer还是需要2轮RPC
    Client   Proposer      Acceptor     Learner
    |         |          |  |  |       |  | --- First Request ---
    X-------->|          |  |  |       |  |  Request
    |         X--------->|->|->|       |  |  Prepare(N)
    |         |<---------X--X--X       |  |  Promise(N,I,{Va,Vb,Vc})
    |         X--------->|->|->|       |  |  Accept!(N,I,V)
    |         |<---------X--X--X------>|->|  Accepted(N,I,V)
    |<---------------------------------X--X  Response
    |         |          |  |  |       |  |  

    唯一的Proposer确定后进入此Proposer的流程，只需要1轮RPC
    Client   Proposer       Acceptor     Learner
    |         |          |  |  |       |  |  --- Following Requests ---
    X-------->|          |  |  |       |  |  Request
    |         X--------->|->|->|       |  |  Accept!(N,I+1,W)
    |         |<---------X--X--X------>|->|  Accepted(N,I+1,W)
    |<---------------------------------X--X  Response
    |         |          |  |  |       |  |

### 流程简述
1. 第一次请求为第N个Proposer竞选，这里可以把Proposer理解为唯一Leader，这一轮RPC和Basic Paxos一样
2. Proposer竞选成功后接下来在N的Proposer任期内的请求都只需要一轮PRC，每次提议I+1

## Fast-Paxos

### 基本成功流程

    Client    Leader         Acceptor      Learner
    |         |          |  |  |  |       |  |
    |         X--------->|->|->|->|       |  |  Any(N,I,Recovery)
    |         |          |  |  |  |       |  |
    X------------------->|->|->|->|       |  |  Accept!(N,I,W)
    |         |<---------X--X--X--X------>|->|  Accepted(N,I,W)
    |<------------------------------------X--X  Response(W)
    |         |          |  |  |  |       |  |

### 有协调参与的冲突恢复。

    注意:协议没有指定如何处理被丢弃的客户端请求。
    Client   Leader      Acceptor     Learner
    |  |      |        |  |  |  |      |  |
    |  |      |        |  |  |  |      |  |
    |  |      |        |  |  |  |      |  |  !! Concurrent conflicting proposals
    |  |      |        |  |  |  |      |  |  !!   received in different order
    |  |      |        |  |  |  |      |  |  !!   by the Acceptors
    |  X--------------?|-?|-?|-?|      |  |  Accept!(N,I,V)
    X-----------------?|-?|-?|-?|      |  |  Accept!(N,I,W)
    |  |      |        |  |  |  |      |  |
    |  |      |        |  |  |  |      |  |  !! Acceptors disagree on value
    |  |      |<-------X--X->|->|----->|->|  Accepted(N,I,V)
    |  |      |<-------|<-|<-X--X----->|->|  Accepted(N,I,W)
    |  |      |        |  |  |  |      |  |
    |  |      |        |  |  |  |      |  |  !! Detect collision & recover
    |  |      X------->|->|->|->|      |  |  Accept!(N+1,I,W)
    |  |      |<-------X--X--X--X----->|->|  Accepted(N+1,I,W)
    |<---------------------------------X--X  Response(W)
    |  |      |        |  |  |  |      |  |

### 无协调者参与的相冲突恢复

    Client   Leader      Acceptor     Learner
    |  |      |        |  |  |  |      |  |
    |  |      X------->|->|->|->|      |  |  Any(N,I,Recovery)
    |  |      |        |  |  |  |      |  |
    |  |      |        |  |  |  |      |  |  !! Concurrent conflicting proposals
    |  |      |        |  |  |  |      |  |  !!   received in different order
    |  |      |        |  |  |  |      |  |  !!   by the Acceptors
    |  X--------------?|-?|-?|-?|      |  |  Accept!(N,I,V)
    X-----------------?|-?|-?|-?|      |  |  Accept!(N,I,W)
    |  |      |        |  |  |  |      |  |
    |  |      |        |  |  |  |      |  |  !! Acceptors disagree on value
    |  |      |<-------X--X->|->|----->|->|  Accepted(N,I,V)
    |  |      |<-------|<-|<-X--X----->|->|  Accepted(N,I,W)
    |  |      |        |  |  |  |      |  |
    |  |      |        |  |  |  |      |  |  !! Detect collision & recover
    |  |      |<-------X--X--X--X----->|->|  Accepted(N+1,I,W)
    |<---------------------------------X--X  Response(W)
    |  |      |        |  |  |  |      |  |

### 无协调者的冲突恢复、角色崩溃的Fast-Paxos

    Client         Servers
    |  |         |  |  |  |
    |  |         X->|->|->|  Any(N,I,Recovery)
    |  |         |  |  |  |
    |  |         |  |  |  |  !! Concurrent conflicting proposals
    |  |         |  |  |  |  !!   received in different order
    |  |         |  |  |  |  !!   by the Servers
    |  X--------?|-?|-?|-?|  Accept!(N,I,V)
    X-----------?|-?|-?|-?|  Accept!(N,I,W)
    |  |         |  |  |  |
    |  |         |  |  |  |  !! Servers disagree on value
    |  |         X<>X->|->|  Accepted(N,I,V)
    |  |         |<-|<-X<>X  Accepted(N,I,W)
    |  |         |  |  |  |
    |  |         |  |  |  |  !! Detect collision & recover
    |  |         X<>X<>X<>X  Accepted(N+1,I,W)
    |<-----------X--X--X--X  Response(W)
    |  |         |  |  |  |
## 学习资源

[https://zh.wikipedia.org/zh-cn/Paxos%E7%AE%97%E6%B3%95](https://zh.wikipedia.org/zh-cn/Paxos%E7%AE%97%E6%B3%95)
[https://www.youtube.com/watch?v=BhosKsE8up8](https://www.youtube.com/watch?v=BhosKsE8up8)
