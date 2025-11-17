---
title: "Kafka如何实现主从同步?"
date: 2025-10-14
categories: ["Kafka"]
tags: ["Kafka","同步"]
draft: false
---
>Kafka 实现主从同步（即 Leader 副本和 Follower 副本之间的数据同步）是其保证数据高可用性和持久性的核心机制。这个过程是**异步拉取（Pull）的，并由 ISR（同步副本集合）机制严格管理。
### 1.异步拉取
__与一些数据库的 Push 模式不同，Kafka 的副本同步采用 Pull 模型：__
~~~
主动方 (Follower)：Follower 副本是主动方。它会不断地向 Leader 副本发送请求，请求拉取新的消息数据。
拉取单位：Follower 拉取的最小单位是 **日志段（Log Segment）**中的一批消息。
这种拉取模式允许 Follower 控制自己的复制速率。如果 Follower 暂时负载过高，它可以减慢拉取速度，避免被 Leader 的高速写入压垮。
~~~

 
### 2.关键同步指标
__Follower 在同步过程中，会维护和使用两个关键的偏移量（Offset）：__
~~~
LEO (Log End Offset)：表示该 Follower 已成功写入本地日志的最新消息的下一个 Offset。
HW (High Watermark)：表示所有 ISR 集合中的副本都已经复制并确认写入的最新消息的下一个 Offset。
重要性：HW 之前的消息对 Consumer 是可见且安全的，而 HW 之后的 Leader 消息对 Consumer 是不可见的，以防 Leader 宕机导致数据丢失。
~~~

### 3.ISR (In-Sync Replicas) 机制的保障
__同步副本集合（ISR）是衡量同步状态的核心机制：__

~~~
Leader 维护 ISR：Leader 副本负责维护 ISR 列表。ISR 列表包括 Leader 自身和所有与 Leader 保持“同步”的 Follower 副本。
同步判断标准：
 -Follower 必须在配置的时间阈值（replica.lag.time.max.ms）内持续向 Leader 发送拉取请求。
 -Follower 的 LEO 必须与 Leader 的 LEO 保持在一个可接受的范围内。
副本移出：如果 Follower 无法满足上述条件（如网络延迟过高、宕机），它会被 Leader 移出 ISR。
数据持久性保证：当生产者（Producer）设置为 acks=all 时，Leader 必须等待 ISR 中的所有副本都确认写入了消息，才会返回 ACK 成功。这确保了只要 ISR 中有一个副本存活，数据就不会丢失。
~~~
### 4.主从同步流程简述
~~~
Follower 发送 Fetch 请求：Follower 向 Leader 发送 Fetch Request，请求从自己的 LEO 开始的新消息。
Leader 发送消息：Leader 从自己的日志中读取从 Follower LEO 开始的消息，并返回给 Follower。
Follower 写入并更新 LEO：Follower 接收到消息后，将其追加写入到自己的本地日志中，并更新自己的 LEO。
Leader 更新 HW：Leader 收到 Follower 的成功响应后，会检查 所有 ISR 副本的 LEO，并更新 HW 为所有副本 LEO 的最小值。
~~~
