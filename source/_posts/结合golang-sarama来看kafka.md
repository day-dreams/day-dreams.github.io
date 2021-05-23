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
* Coordinator 管理一个ConsumerGroup的状态，主要是offset消费情况
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


## Sarama/SyncProducer

作为Proudcer，各组件之间的关系非常清晰，基本是通过goroutine进行异步交互。上层的Proudcer()，经过层层dispatch之后，会触发真正的Produce请求，给到对应的leader broker。

![producer](/pics/kafka_producer_sarama_brokers.png)

以`AsyncProducer`为例，看看Producer的生命周期里，都发生了哪些事情：
* NewAsyncProducer，初始化Producer
  * NewClient
    * 解析配置，从集群列表，随机挑选一个broker，去拉取整个kafka集群的元数据：Topic列表、Partition列表、Parition的Leader/Follower列表
    * 起一个goroutine，异步更新kafka集群的元数据 
  * newAsyncProducer，初始化channel
    * newTransactionManager，向Kafka集发起注册请求，成为一个Producer
    * 起好几个channel，分别用于input、result、error 
    * 起dispatcher goroutine。从input channel获取消息，投放到合适的分区leader
      * 每次收到一个代生产的消息，都先用Client去集群里查询该Topic/Partition的Leader
      * 拿到Leader后，向Leader写入新消息，并根据配置，等待集群ACK
    * 起retry goroutine
      * 有的时候，消息生产可能会失败，专门搞了一个retry channel，存储需要retry的消息
      * 无脑的从这个retry channel取出消息，然后塞到input channel，触发消息生产

## Sarama/ConsumerGroup

* NewConsumerGroup

## 我用到的封装版本

在内部rpc封装的背景下，我消费kafka消息，实际上是封装了一层。 sarama库要求使用方实现一个consume client，嵌入到sarama的框架。

```golang

// ConsumerGroupHandler instances are used to handle individual topic/partition claims.
// It also provides hooks for your consumer group session life-cycle and allow you to
// trigger logic before or after the consume loop(s).
//
// PLEASE NOTE that handlers are likely be called from several goroutines concurrently,
// ensure that all state is safely protected against race conditions.
type ConsumerGroupHandler interface {
	// Setup is run at the beginning of a new session, before ConsumeClaim.
	Setup(ConsumerGroupSession) error

	// Cleanup is run at the end of a session, once all ConsumeClaim goroutines have exited
	// but before the offsets are committed for the very last time.
	Cleanup(ConsumerGroupSession) error

	// ConsumeClaim must start a consumer loop of ConsumerGroupClaim's Messages().
	// Once the Messages() channel is closed, the Handler must finish its processing
	// loop and exit.
	ConsumeClaim(ConsumerGroupSession, ConsumerGroupClaim) error
}

```

我用的封装大概是这个样子:

* 死循环监听msg chan
  * 把消息放到buffer
  * 每X条消息，批量消费一次buffer
  * 每Y秒，批量消费一次buffer
  * 如果消费成功，标记ok
  * 如果失败，直接返回，等待下次重新消费消息
  
```golang
func (c* client)ConsumeClaim(session* ConsumerGroupSession, claim *ConsumerGroupClaim)error{
  buffer:=map[int]msg{}
  ticker:=ticker.New(time.Second*5)
  for {
    select {
      case msgs :<- claim.msgs:
        for _,msg := msgs {
          if len(buffer) > X { // 每X条数据集中消费一次，有错误就返回，下次重新消费整个批次的数据
            err:= doLogicHandle(buffer)
            if err!=nil{
              return 
            }

            session.MarkOk(buffer) // 消费成功
          }

          buffer[len(buffer)]=msg
        }
      case _ = <- ticker.C: // 定时消费一批数据
        err:=doLogicHandle(buffer)
        if err!=nil{
          return
        }

        session.MarkOk(buffer) // 消费成功

        delete(buffer)
    }
  }
}

```


## 实战经验之谈

* 消息体太长，会发生截断吗？如果截断，上层应用程序怎么处理？
* kafka分区数能改变吗？如果改变了，怎么保证分区顺序一致；如果不能改变，某个分区挂了怎么办
* consumer rebalance是怎么发生的？最终会如何影响consumer group？