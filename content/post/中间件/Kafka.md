+++
author = "FarSightLi"
title = "kafka基础架构理论"
date = "2025-06-05"
description = "kafka基础架构理论"
categories = [
 "Kafka"
]
tags = [
 "Kafka"
]

+++

## Kafka要解决的问题：

原视频：

【消息队列Kafka是什么？架构是怎么样的？5分钟快速入门】https://www.bilibili.com/video/BV1TT421y79S?vd_source=f2fbf19a95f21a30c8f56673d2b182b7

消费者与生产者之间的生产速度和消费速度是不匹配的。如果生产的消息都往消费者塞，那么消费者很容易挂掉。那么我们可以把消息发往一个消息队列。消息队列可以起到 1削峰填谷 2.解耦 3.异步 等作用。但如果消息队列与消费者耦合，会有很多问题。于是需要把消息队列抽象出来，而Kafka就是消息队列的一种中间件。

随着消费者和生产者的增加，消息的吞吐增加了。然而吞吐量到达一定程度，那么多个消费者/生产者就会共同争抢一个消息队列，会有等待时间。这样就出现了性能问题。那么这时候就可以增加消息队列，指定不同种类的消费者和生产者使用某一个消息队列--这个就是topic。单个topic数据量过大时，那么就可以将单个topic拆开，这就是partition，然后每个消费者消费不同partition。

如果多个partition都放在同一台机器上，则会对机器造成性能压力，那么可以将不同partition放在不同机器上，那么每个机器就是一个borker。

如果broker的机器挂掉了，那么数据就全丢了。为了做到高可用性，往往将partition做备份即副本replicas。

leader负责读写、follower负责同步leader。为了以防万一，往往还会对数据进行持久化。将数据写到磁盘上，然后根据某种策略清理磁盘

### 消费组 consumer group相关概念

消费组是为了解决：单个消费者消费速度跟不上生产者，那么就增加多个消费者组成消费组来共同消费某个topic。

kafka控制了在一个消费组中，，同一 Partition 内的消息在同一个 Consumer Group 中只会被一个消费者实例处理。而不同消费组之间是可以独立的消费同一条消息。

消费组订阅的是topic，订阅后，调度者coordinate就会将不同partition分配给不同消费者实例。调度员coordinate会在消费者增减、partition增减、消费组订阅新topic时进行重新分配rebalance。rebalance时group中的所有消费者会暂停消费至重新分配完毕

### 生产者的核心机制

消息分发策略：生产者要指定topic，那么进入topic的消息应该去哪个partition？这有不同策略，比如轮询分配，比如按某个规则hash分配。

ACK机制：可以不确认（acks=0）、leader确认（acks=1）、leader及follower确认（acks=all）

acks=all时，还有一个参数min.insync.replicas即isr的副本大于等于这个参数时，就可以向生产者返回ack



发送时，消息先放到发送缓冲区中，然后再攒批聚合发送

### 高水位线  high watermark机制

hw标记了一个位置--所有副本都已经同步的消息offset位置，消费者只能读取到小于hw的消息

AI画的图：

| Offset | 0   | 1   | 2   | 3   | 4   | 5   | 6   |
| ------ | --- | --- | --- | --- | --- | --- | --- |
| 消息     | A   | B   | C   | D   | E   | F   | G   |
| 状态     | ✅   | ✅   | ✅   | ✅   | 🚫  | 🚫  | 🚫  |

```
                                          ↑
                                      HW=4 (安全水位线)
```

此时当所有follower确认消息E持久化后，HW就会更新为5

HW只由Leader进行更新维护，会由kafka的各种强制刷盘机制写到磁盘上

疑问：

1.为什么有topic和partition的概念，直接新增消息队列不就行了？

topic是指Kafka将一些生产、消费的数据抽象出来，方便不同功能的生产者消费者操作消息。也就是起到分类功能。还可以使生产者消费者只关心如何收发数据，生产者不必额外设计标识使消费者正确消费。这是一种逻辑上的分区。

而partition将topic分开，这种分区是一种物理上的分区。不同partition分布在不同broker或磁盘上，使减少读写io的冲突。此外，partition还提供了并发消费的能力。

2.假如副本flower尚未拉取同步完leader的消息，lead挂掉了，此时选举出来的leader岂不是会丢消息？

这是AI给的场景复现：

1. 初始状态
   
   * Topic 分区 P0 有 3 个副本：Leader L1，Follower F2，Follower F3（ISR = [L1, F2, F3]）
   
   * 生产者发送消息 M1 到 L1，L1 将 M1 写入本地日志（但尚未同步给 Follower）

2. `故障发生`
   
   * L1 突然宕机（此时 M1 未同步给任何 Follower）
   
   * 系统需选举新 Leader（F2 或 F3）

3. 风险
   若选出的新 Leader 是 F2 或 F3，它们从未收到过 M1，导致 M1 永久丢失！

解决方案：1、ACK策略指定all 则只要生产者确认成功，那所有ISR都有数据，不会丢失

2. 假如acks=1，那么在此场景中，
   
   * Leader (L) 已写入消息 E (Offset=4)
   
   * F1 同步了 E → ISR = [L, F1, F2]
   
   * F2 未同步 E（因网络延迟）
   
   * 此时 HW 仍为 4（因 F2 未确认，消息 E 不算安全）
   
   * 突发故障：Leader (L) 宕机！
     
     F1被选举为Leader，会进行检查，发现HW=4，则将消息E丢弃。而F2检查后不做处理（因为符合HW）。然后就是正常的同步数据流程
     
     

### 消费者的核心机制

TODO



### 从代码看不同acks处理

以下代码由AI提供，暂未验证是否准确

```java
// --------------------------
// CASE 1: acks=0 (最高吞吐)
// --------------------------
props.put("acks", "0"); // 无需确认
producer.send(new ProducerRecord<>("topic", "key", "value"));
// 特点：发送即忘，不处理响应（可能丢失）

// --------------------------
// CASE 2: acks=1 (平衡模式)
// --------------------------
props.put("acks", "1"); // Leader 确认
producer.send(new ProducerRecord<>("topic", "key", "value"), (metadata, e) -> {
    if (e != null) log.error("发送失败", e); // 处理失败
    else log.debug("发送成功: {}", metadata.offset());
});
// 特点：处理Leader响应，网络异常可能重复

// --------------------------
// CASE 3: acks=all (高可靠)
// --------------------------
props.put("acks", "all"); 
props.put("min.insync.replicas", "2"); // 要求至少2个副本同步
producer.send(new ProducerRecord<>("topic", "key", "value"), (metadata, e) -> {
    if (e != null) {
        if (e instanceof NotEnoughReplicasException) {
            // ISR副本不足时的特殊处理
        }
        retryOrFail(e); // 自定义重试逻辑
    }
});
// 特点：严格一致性保障，需处理副本同步异常
```

可以看到，在acks=0的情况下，是没有回调方法的，所以也就拿不到元数据和异常信息（当然，offset=0时默认发送成功），而在acks=1或all的情况下，生产者是可以拿到发送成功后的offset等数据，假如发送失败也可以拿到拿到异常原因，但是发送异常后是否重发或其他逻辑，其实是生产者的代码自定义的，kafka对此并无要求？
