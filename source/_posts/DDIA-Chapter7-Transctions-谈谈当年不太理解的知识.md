---
title: DDIA Chapter7 Transctions 谈谈当年不太理解的知识
date: 2024-02-14 21:46:45
tags: DDIA
---

年少时阅读mysql文档，被事务机制困扰了不少。总觉得是一种死记硬背的知识，现在发现并不是。

# Transactions

## ACID 基本概念

事务由一系列读写操作组合在一起，主要关注数据库能给这些读写操作提供什么保证。

* Atomic 原子性
    * 保证读写操作必须全部执行成功，或者全部失败（被回滚 Rollback/Abort，对db没有任何影响）
* Consistence 一致性
    * 保证数据状态完好无损。
    * 一方面，db可以保证数据取值遵循约定好的规则（长度/大小等），如果违反规则，拒绝存储
    * 另一方面，db的使用者（Application）必须保证，数据写入之后，整体状态没有损坏。比如Application约定只能有10个获奖者，结果自己写入了11个，这是违反一致性的。
    * **这个特性更多偏向Application，跟DB没什么关系。db只能保证基本的数据类型不被违反，但是不理解数据的含义；需要Application自己定义什么是“状态完好无损”。**
* Isolation 隔离
    * 关注不同事务之间的读写操作如何相互影响，这也是我们最感兴趣的和最复杂的点。
* Durability 持久性
    * 保证数据被db完毕后，可以一直保存不出问题。
    * 其实这是不可能的，db只能在有限的条件下保证：如果磁盘/电源等不出问题，那么单机数据永不丢失；如果集群节点大部分可用，那么数据在集群范围内可用。

## Weak Isolation Levels 隔离级别-前人的一些努力
 
其实主要就是探讨Isolation。db为了保证隔离性，需要做出很多努力，才能达到一定程度的隔离性；不同事务的读写操作会相互影响，在不同的隔离级别下带来不同的问题。

* level0 什么也不做
    * 完全没有隔离可言。一个事务的写操作可以被另一个事务里的读操作观察到，Application将拥有一个不稳定的db，随时爆炸。
* level1 Read Committed 读已提交
    * 针对level0，事务A读取到了事务B的数据（Dirty Read，脏读），我们自然想到要对事务A屏蔽事务B的写操作。这很好实现，经典做法就是保存数据的2个版本：事务A把数据从version 1更新到version 2后，只要事务A未提交，事务B就只能读取到version 1，而不是version2；等到事务A提交完成，db在把version1干掉，只保留version。这个机制叫做**Snapshot version of old data**。
    * 针对两个事务的写操作，还要保证它们不要相互影响。典型场景是，事务A/B分别要把2个字段更新为Value1/Value2 Value3/Value4，如果它们互不影响，最终的数据应该是Value1/Value2 or Value3/Value4，两种可能；但是犹豫更新顺序、时机不同，最终结果可能是A/B混合在一起，比如Value1/Value4 or Value3/Value2，此时数据已经坏掉了。这种称为脏写(Dirty Write)。一个简单的解决办法是加锁，比如Mysql/Innodb有row locks，写数据时获得锁，事务完成后释放锁，如此就不会发生Dirty Write。
* level2 Repeatable Read 可重复读
    * level1是有缺陷的，比如事务A第一次读取到了数据x，但是事务B把x更新为y并完成提交，此时事务A再去读取，会获得数据y。此问题成为不可重复度（Unrepeatable Read），数据变成非常不稳定的状态。类似level1引入的snapshot，我们再引入多个版本，只要不主动修改，每个事务里读取的数据总是不变。即多版本控制，（MVCC Multi Version Concurrency Control）
    * MVCC通常这样实现：事务带有自增版本号txid，每条数据加上createdby/deletedby两个字段，承载修改/删除当前数据的txid。**查询数据时，通过比较当前事务和数据的createdby/deletedby的大小，决定是否可见。**比如txid13写入了数据a，txid15把a更新为b，那么txid14可以读取到a（因为txid13小于txid14），但是txid14不能获取到数据b（因为txid14小于txid15）。事务提交后，db后台进程会自动回收不需要保存的数据版本。

看起来level2已经很完美了，但是非常遗憾，还有一些未解决的问题。比如：
    * LostUpdates 两个事务并发的写操作，到底谁生效？
        * 譬如两个事务都想把一个counter字段+1，但是因为大家的逻辑都是```select counter;got value x;update counter=x+1```，即使已经有了row locks，两个事务执行完毕后，counter还是会变为counter+1，而不是counter+2。这种问题可以通过Atomic Write Operations解决，简单来说就```update counter=counter+1```，两个事务个执行一遍，最后结果变为counter+2。
        * 对于非counter的场景，比如事务A想把角色a移动到x位置，事务B也想把角色b移动到x位置，他们的逻辑是```select x;check if x is empty;then move a/b to x```。最终结果是A/B都以为成功移动了角色，实际上只有其中一个成功移入到x。这种逻辑可以概括为，查询数据-判断-写入，第三步写入会影响第一步的查询结果。也就是说当我们要查询某个数据然后决定是否修改时，要防止其他事务也做类似的操作。这时就需要引入**显示锁（Explicit Locks）**，也就是```select x for update```，在事务A执行第一步查询数据时，直接加锁，这样事务B就需要等待了。
    * Write Skew and Phantoms

## Serializability 可串行化-新的思路待检验
