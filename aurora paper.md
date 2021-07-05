---
title: aurora paper: understand and think
tags: 新建,模板,小书匠
category: /小书匠/日记/2021-07
renderNumberedHeading: true
grammar_cjkRuby: true
---

# 创新之处
### 内容
- log is database - redo log作为db
- 计算存储分离 - redo log作为存储，从原体系中独立出来。redo log会materilize redo log 为data page
- 计算部分只写IO数据到存储部分的redo log，减少网络延迟

### 原因
- 对于cloud native环境，由于各计算部件、存储部件的虚拟化，网络延迟成为db on clound上影响性能的主要因素。


# 具体设计
### replication
- 多副本的复制协议：quorom投票选举 - 每个node进行投票。根据得票，决定此次operation是否可行
- 

### segment


### Redo Log
###### 结构特性
- 有序、顺序、形成log chain
- 每个Log Record都有一个自增序号：LSN
###### terminologies
- VCL - Volume Completed LSN: LSN小于VCL的redo log record都是是可用的
- MTR - Mini Transactions: 事务执行的最小单位。每个mini-transaction都由多个连续的log record（根据需要）组成
- CPL - Consistency Point LSN: 一个MTR内的最大VCL。因此CPL其实也表示了一个MTR内的最高VCL。为保证事务的原子性，此MTR内低于此LSN的record不能被截断
- VDL - Volume Durable LSN: volume中小于或等于VCL，且已经成功durable的最大CPL。VDL表示了数据库已经持久化了的处于一致状态的最新位点

### 存储
###### principles
- 尽量减少写入和读取延迟时间
- 将大部分的处理操作都移至后台，增强处理性能 

###### 数据一致性
- 利用gossip协议，在所有peer间进行log record共享和复制，达到数据一致性的目的
- 没有使用2PC等一致性协议，因为2PC的容错性较差，在大型分布式环境中不适用

###### 事务一致性
数据库系统中有很多事务，在发生故障时，怎么决定那些TR要丢弃，哪些TR要提交？
- 为了保证不破坏mini-transaction原子性，所有大于VDL的日志，都需要被截断。

###### 在存储节点上进行写操作
 - 接收redo log recored并将其添加到内存队列中
 - 将redo log record 持久存储到磁盘上并将确认消息发送给数据库实例；
 - 将受到的redo log record进行组织并识别出由于某些batch丢失而导致的redo log record空洞。
 - 通过gossip协议与其他的storage node同步，获得丢失的redo log record，进而填补确实的空洞。
 - 将redo log record合并到新的data page中。
 - 定期将阶段性的redo log和对应的新的data page暂存到S3。
 - 定期对旧版本数据进行GC。
 - 定期验证data page上的CRC code。
   

![write process in storage node](https://pic2.zhimg.com/80/v2-4ec87938c5171ee37374f863af320aa9_720w.jpg)

######  在存储节点上进行读操作
- 如“数据一致性”一节所述，aurora通过在storage node peer间共享同步数据达到一致性的目的。也因此，正常的node都处于已同步或同步中状态。若不要求绝对的即时查询结果，读可以直接在单分片上进行，不必依据quorom协议

- 例外：若故障恢复的时候data page需要rebuild，这时丢失了运行时状态，才需要基于quorum进行读取。

- 事务的一致性 - TBC，故障恢复时，需要决定那些事务是需要commit，哪些需要rollback

###### 写操作与Redo Log

###### 读操作与Redo Log


###### cache



### 故障处理
- 数据库服务奔溃
- 存储服务奔溃
# 优势
- 存储计算分离：构建跨多中心、自我管理、容错、自我恢复、独立的存储服务，与上层database service故障分离
- database service仅需要写入redo log到storage service，大幅减少网络IO延迟，增加系统性能
- 故障隔离：隔离因check point等后台服务引起的性能抖动，增强database service的性能稳定性
- backup/restore：TBC