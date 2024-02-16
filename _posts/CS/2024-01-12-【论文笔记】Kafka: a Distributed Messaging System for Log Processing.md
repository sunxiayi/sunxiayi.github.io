---
title: 【论文笔记】Kafka a Distributed Messaging System for Log Processing
date: 2024-01-12
categories: CS
tags:
- 论文
- 系统设计
- scaling
---

结合论文以及youtube上找到的一些信息写个总结（which意味着以下信息结合了两者，不只有论文）。

## Kafka简介

- 是什么：高性能的分布式日志流消息系统
- 解决了什么问题：实现线上（realtime）和线下（offline）对日志流的处理
- 创新性和独特性：适用于大规模、高通量、不介意少量数据丢失的情况，拉（pull）模式，可有多个客户端重复读取数据。


## 设计与架构

每个broker负责一些topic，每个topic里包含着一些partition，每个partition里包含着一些message。客户端从每个分区中顺序读取消息，但各个不同分区的信息不是顺序读取的。几个消费者构成消费组，每个消费组可以订阅一到多个主题，每条消息在一个消费组内只会传给其中一个消费者，各个消费者之间相互独立不互相影响。

- 储存：数据被储存在文件中，在内存中有一个索引，以第一条消息为键来查找文件
- 控制层：
  - 在这篇论文中，分布式架构中的node增减，由Zookeeper来实现对broker, consumer, ownership, offset的追踪。其中，对broker记录其主机和端口，以及它所包含的topic, partition。对consumer记录其所在的组，以及它订阅的topic。每个consumer group都有一个ownership记录和offset记录。ownership记录里会记录哪个consumer正在读取哪个partition，offset则负责记录读到了哪个offset。因为broker和consumer都是随时可能会坏掉的机器，broker, consumer和ownership的值都是暂时的，而offset由于需要维持其在同一个consumer group的递增性，是一个永恒（persistent）的值。当consumer或者broker有成员变动的时候，zookeeper会通知每个consumer。
  - 从网上更新的信息上看，Zookeeper已经被Raft替代了。主要原因是系统的简易性和提升效率。leader selection都遵循raft的算法。metadata都存在子节点的内存中（可以是broker也可以是单独的controller）。
- 数据层：在本论文写作时，数据复制被列为未来的“发展方向”，从今天看来kafka已经实践了数据复制。从网上的信息来看，利用offset和watermark来标记committed的记录。leader failure时，从in sync replica中选择一个replica作为新的leader，并更新epoch，follower根据fetchRequest中的epoch来更新自己的log（建议看视频，视频中说的比较清楚）。
- 在[消费者组](https://www.youtube.com/watch?v=ovdSOIXSyzI&list=PLa7VYi0yPIH14oEOfwbcE9_gM5lOZ4ICN&index=7)中，通过有冗余的group coordinator来实现同组分配，有range, round robin, sticky等分区形式；优化了消费者rebalance的过程，使得系统的传输效率不那么受rebalance的影响。
- 交付保证：在论文中提到的at-least-once delivery在后面进行了[提升](https://www.youtube.com/watch?v=Ki2D2o9aVl8&list=PLa7VYi0yPIH14oEOfwbcE9_gM5lOZ4ICN&index=10)，提供transaction和exactly once的交付，主要是引入了一个coordinator来进行2PC。


## 性能提升

- 缓存设置：在Kafka，没有在内存中设置缓存，而利用了文件系统的页面缓存（page cache）。因为顺序读写，页面缓存是一个增加性能的选择，可以减少进程的缓存，并在重启时无需被重建，还减少了GC的压力（kafka用java写的，java的gc在堆内数据增多时性能变差）。

- 网络传输
  - 生产者和消费者都进行批消息处理
  - 通常传输文件到socket需要经过4个步骤：1）从储存系统读取文件，放到page cache中；2）从page cache复制数据到应用层；3）将数据从应用层复制到网络另一端的kernel层；4）从kernel传输数据到socket。Kafka利用了sendfiel API，跳过了步骤2和3，增加了效率。


## Ref
- [原文地址](https://notes.stephenholiday.com/Kafka.pdf)
- [Apache Kafka Architecture](https://www.youtube.com/playlist?list=PLa7VYi0yPIH14oEOfwbcE9_gM5lOZ4ICN)