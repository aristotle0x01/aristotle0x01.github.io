---
layout: post
title:  "zookeeper-2-zab协议"
date:   2021-05-17
categories: zookeeper zab leader
---
# zab协议

## zab是什么？

全名：zookeeper atomic broadcast protocol。如果说zookeeper是一个软件系统，那么zab可谓系统设计文档，描述和定义了系统功能，尤其内部逻辑流程，状态转换，是zookeeper的中枢神经系统

## 为什么要有zab?

众所周知，zab之前已有paxos，但是paxos晦涩难懂，各种实现理解不一致，且直接使用也不能满足系统设计需求。所以yahoo设计了zab，要解决以下核心问题：

1）leader election

2)   状态变更同步一致性（高可用多副本带来的问题）

3）高读写吞吐能力

## 节点状态

任意一个zookeeper server节点都可能处于三种可能的状态：

```
{
election,  // 选举状态；系统初始或者leader节点挂掉，集群重新选主时
following, // 从属状态，表示节点从属于其它主节点
leading    // 主节点状态，表示节点处于领导地位
}
```

绝大多数正常情况下，对于一个集群来说：都只有一个节点处于leading状态，其余所有节点处于following状态，election只是发生选主时的临时状态

## 三个阶段

绝大多数情况下，系统都处于broadcast阶段(完成选主的稳定状态)，正常接受外部读写请求。

从完整描述系统状态变化的角度，大多少理论文献将其状态流转划分为三个阶段：**discovery->synchronization->broadcast**

![3 phases](https://user-images.githubusercontent.com/2216435/118471675-96262300-b73a-11eb-8dfc-51e7bb8fbd17.png)

这三阶段有个前提，就是已经找出了唯一潜在leader(prospective leader)，这个部分大部分文献没将，后面再说

### discovery

这个阶段的任务是从所有节点中找出包含最近的已提交历史，并将之赋予prospective leader。换言之，这个阶段完成时，prospective leader将拥有最全最新的已提交数据，并建立新的王朝(newepoch)

### synchronization

这个阶段的任务就是将discovery阶段leader的提交历史同步给其它节点。在这个阶段完成后，至少quorum个节点在之前已经提交的数据历史上达成了一致，此时新皇正式登基(newleader)

### broadcast

只有前两个阶段完成，leader节点才能开始接受外部数据变更请求，开始广播处理。节点不挂的情况下，系统将长期愉快的处于这个阶段和状态

## yahoo真正实现的阶段

![real phases](https://user-images.githubusercontent.com/2216435/118475863-6299c780-b73f-11eb-8b32-f8a2ca9816c4.png)

实际上yahoo在实现的时候，将选择潜在leader(phase-0)和discovery(phase-1)合二为一，于是便有了下面三个阶段：

- fast leader election
- recovery phase
- broadcast phase 

在完成阶段1(fast leader election)后，潜在leader已经具备最全最新的已提交变更历史了

## 如何找出候选leader(phase-0)？

对于每个节点，都有一个标识sid，假设为整数

对于每个节点，都有最新提交历史序列号zx，由(epoch:counter)组成，counter为某个epoch内的自增序列

对于每个节点，都优先推选自己，并且将消息发送给其它节点

那么上述描述的核心即为互相比较接受并交互：**（zx, sid）->  (epoch:counter, sid)**的大小

规则为：**zx大者为优先，若zx相同，那么sid大者为优先，得到quorum数量投票的节点为leader**

![](https://user-images.githubusercontent.com/2216435/118481800-75fc6100-b746-11eb-96a4-4038d56390e9.png)

## zab变更提交的顺序性保证

### local order

对于某个特定的epoch，其leader是特定且唯一的，那么通过tcp协议的顺序性，假设**zx1 < zx2**(变更提交序列号)，那么zx1一定先于zx2完成提交

### primary order

对于两个epoch，**e1 < e2**, 假设zx1在e1时段提交，zx2在e2时段提交，那么zx1一定先于zx2提交。这是由于e2时段的leader(在synchronization阶段)必须完成所有之前阶段的提交，才能开始e2时段的变更请求

## 变更请求的处理(broadcast phase)

此时leader接受客户端变更请求，利用zab协议，将数据同步到quorum节点

![](https://user-images.githubusercontent.com/2216435/118478213-303d9980-b742-11eb-8efc-18c14cda4d4c.png)



![](https://user-images.githubusercontent.com/2216435/118478322-4fd4c200-b742-11eb-84a9-3eb806384d70.png)

## 重要参考

[ZooKeeper’s atomic broadcast protocol: Theory and practice](http://www.tcs.hut.fi/Studies/T-79.5001/reports/2012-deSouzaMedeiros.pdf)

[Zab: High-performance broadcast for primary-backup systems](https://marcoserafini.github.io/papers/zab.pdf)

[Apache Zookeeper Internals](https://ssudan16.medium.com/apache-zookeeper-internals-7b063f9d74ac)

