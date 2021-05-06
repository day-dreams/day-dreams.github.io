---
title: 结合golang sarama来看kafka
date: 2021-04-29 00:33:45
tags:
---

是的，kafka的设计非常经典。大概半年前，我总结了一次kafka的细节，但是现在已经全部忘记。只是依稀记得几个关键词：partition broker leader。

我的工作内容是在是简单，几乎不用去关注kafka的特性。无脑使用就完了。

这次相结合kafka源码、golang saram库，来回顾kafka的特性。

## Partition和Broker

## Producer

## Consumer/Consumer Group

## 实战经验之谈

* 消息体太长，会发生截断吗？如果截断，上层应用程序怎么处理？
* kafka分区数能改变吗？如果改变了，怎么保证分区顺序一致；如果不能改变，某个分区挂了怎么办