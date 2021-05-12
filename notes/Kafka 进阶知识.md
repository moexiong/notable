---
attachments: [Clipboard_2021-05-12-23-10-24.png, Clipboard_2021-05-12-23-17-37.png, Clipboard_2021-05-12-23-20-15.png, Clipboard_2021-05-12-23-23-01.png, Clipboard_2021-05-12-23-41-40.png]
title: Kafka 进阶知识
created: '2021-05-12T14:36:42.827Z'
modified: '2021-05-12T16:26:36.801Z'
---

# Kafka 进阶知识

## Kafka 名词解释
> - HW(High Watermarker)：高水位线。所有HW之前的数据都理解为是已经备份的，当所有节点都备份成功，Leader会更新水位线。
> - ISR(In-Sync-Replicas)：正在同步中的副本。Kafka的Leader会维护一份处于同步的副本集合，如果在`replica.lag.time.max.ms`时间内系统没有发送fetch请求，或者依然在发送请求，但是在该限定时间内没有赶上Leader的数据就会被踢出ISR列表。(0.9.0后已废弃的配置：`replica.lag.max.messages`消息个数限定，这个会导致其他Broker节点频繁加入和退出ISR)
> - LEO(Log End Offset)：日志标识。log and offset标识的是每个分区中最后一条消息的下一个位置，分区中的每个副本都有自己的LEO。

### Kafka 高水位
Kafka中的Topic被分为多个分区，分区是按照Segments存储文件块。分区日志是存储在磁盘上的日志序列，Kafka可以保证分区里的事件是有序的。其中Leader负责对应分区的读写，Follower负责同步分区的数据。但是在**0.11前，使用HW保证数据的同步，但是基于HW的数据同步可能会导致数据的不一致或者是乱序**。

消息同步方式：
![消息同步方式](@attachment/Clipboard_2021-05-12-23-17-37.png)
高水位的更新：
![高水位的更新](@attachment/Clipboard_2021-05-12-23-10-24.png)
> 1,2...：代表了上图中的segment。
> HW：就是所有副本都同步的公共的位置。
> LEO：每个副本的最后一个同步位置。

消息丢失案例（0.11以前的版本，使用需谨慎）：
![消息丢失案例](@attachment/Clipboard_2021-05-12-23-20-15.png)
> 1. 副本同步Leader数据m2时，还没有更新自己的HW。
> 2. Leader确认副本已经同步过m2，所以主机更新了自己HW=1。
> 3. 副本突发故障，重启后发现自己的HW=0，丢弃1位置的数据。
> 4. 副本准备连接Leader确认自己的水位线时，Leader也故障。
> 5. 副本被选为Leader，导致1的位置数据变为m3，从而丢失了m2数据。

数据不一致问题（0.11以前的版本，使用需谨慎）：
![数据不一致问题](@attachment/Clipboard_2021-05-12-23-23-01.png)
> 1. 副本数据只同步到m1，HW=0，Leader的数据写到m2，HW=1。
> 2. 副本和Leader同时故障。
> 3. 副本先行重启，旧Leader没有重启，副本被选为新Leader。
> 4. 新Leader收到一条记录m3写入，更新HW=1。
> 5. 旧Leader重启，向新Leader同步数据，发现HW=1，认为已经同步，实际数据不一致。

Epoch解决高水平截断问题（0.11+之后的改进）
![Epoch解决高水平截断问题](@attachment/Clipboard_2021-05-12-23-41-40.png)
> 1. 控制器controller负责管理epoch信息，并存储在zookeeper中。
> 2. 控制器将epoch信息同步给Leader，也就是LeaderEpoch。
> 3. 每次Leader接收到Producer的消息时，使用LeaderEpoch标记Message。
> 4. 副本主动同步获取LeaderEpoch编号，替换HW的标记，作为消息的截断。
> 5. 该epoch信息还会随着每一次LeaderAndIsrRequest传递给每一个新的Leader。
>
> 改进点：消息格式改进，每个消息集都带有一个4字节的Leader Epoch号。在每个日志目录中，会创建一个新的Leader Epoch Sequence文件，在其中存储Leader Epoch的序列和在该Epoch中生成的消息的Start Offset。它也缓存在每个副本中，同时还缓存在内存中。
>
> Follower -> Leader：首先将新的Leader Epoch和副本的LEO添加到Leader Epoch Sequence序列文件的末尾并刷新数据。给Leader产生的每个新消息集都带有新的Leader Epoch标记。
>
> Leader -> Follower：如果需要从本地的Leader Epoch Sequence加载数据，将数据存储在内存中，给相应的分区的Leader发送Epoch请求，该请求包含最新的EpochID，Start Offset信息(历史信息)。Leader接收到信息以后返回该EpochID所对应的Last Offset信息。该信息可能是最新的EpochID的Start Offset或者是当前EpochID的Log End Offset信息。

## Kafka Eagle
Kafka监视系统，在github上直接安装Kafka-eagle-web就可以。
[Kafka-Eagle Github](https://github.com/smartloli/kafka-eagle/)


## 小结
1. 高水位截断的问题？
> 主要是0.11版本之前，会发生数据丢失和不一致的问题。主要发生在同步过程。（0.11+版本epoch已解决）
