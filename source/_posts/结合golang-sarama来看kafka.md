---
title: 结合golang sarama来看kafka
date: 2021-04-29 00:33:45
tags:
---

是的，kafka的设计非常经典。大概半年前，我总结了一次kafka的细节，但是现在已经全部忘记。只是依稀记得几个关键词：partition broker leader。

我的工作内容是在是简单，几乎不用去关注kafka的特性。无脑使用就完了。

这次相结合kafka源码、golang saram库，来回顾kafka的特性。

## 一些概念

结合源码，看看有kafka进行了哪些抽象。 浅尝辄止。

### Broker拓扑图

* Topic 没啥好说的。
* Particion 一个topic下，有好几个分区，供好几个consumer、producer来操作
* Broker  代表一个kafka节点。每个节点可以有很多Patition
* 一个Topic的数据，可以分散在多个Partition。
  * 为了保证可靠性，每个Partion有1个Leader和若干个Replicas。
  * 所以会出现这样的拓扑图
    * Broker1是某个topic的某个Partition leader
    * 同时Broker1有和其他几个Broker一样，都是某个Topic的某个Partition replicas

![Broker拓扑图](/pics/kafka_broker_topic_partition.png)

### Partition Leader选举和复制备份

基本上，我们关注的就是ISR, InSyncReplicas。
  * Partition具有1个LeaderBroker+N个FollowerBroker，生产消息时，数据被先写入到Leader，然后等待Follower来主动拉取新数据
  * 如果Follower拉取数据的速度比较快，可以跟Leader保持数据一致，那么被称为InSyncReplicas(ISR)
  * ISR保障了数据可靠性，Leader上的数据总可以在某个ISR里找到。所以即使Leader挂了，ISR也可以马上顶上去，称为新的Leader
  * 根据Follower拉取数据的情况，Leader可以判断Follower的状态
    * 如果10s内没有来拉过数据(还有其他条件)，Follower很可能已经落后于Leader了。此时Consumer不会再从这个落后的Follower拉取数据
  
ISR集合代表了某个分区的最新状态，Consumer只从ISR集合里拉取数据。

![isr](/pics/kafka_isr.png)

* ReplicaManager,作为Broker的核心逻辑之一，负责管理分区状态
  * 周期性的执行两个逻辑
    * `isr-expiration` 遍历当前Broker的每个partition，检查同步状态，如果Broker已经落后Leader很多了，就把broker从Partition的ISR集合里踢出去
    * `shutdown-idle-replica-alter-log-dirs-thread`




## Consumer/Consumer Group

## 实战经验之谈

* 消息体太长，会发生截断吗？如果截断，上层应用程序怎么处理？
* kafka分区数能改变吗？如果改变了，怎么保证分区顺序一致；如果不能改变，某个分区挂了怎么办