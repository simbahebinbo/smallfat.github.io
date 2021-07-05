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


### 存储
###### priciples
- 尽量减少写入和读取延迟时间
- 将大部分的处理操作都移至后台，增强处理性能 

###### 写
 - 接收redo log recored并将其添加到内存队列中
 - 将redo log record 持久存储到磁盘上并将确认消息发送给数据库实例；
 - 将受到的redo log record进行组织并识别出由于某些batch丢失而导致的redo log record空洞。
 - 通过gossip协议与其他的storage node同步，获得丢失的redo log record，进而填补确实的空洞。
 - 将redo log record合并到新的data page中。
 - 定期将阶段性的redo log和对应的新的data page暂存到S3。
 - 定期对旧版本数据进行GC。
 - 定期验证data page上的CRC code。
   

![write process in storage node](https://upload-images.jianshu.io/upload_images/7111776-ffc117cda46b3577.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

###### 读
###### cache
### 数据一致性
### 故障处理
- 数据库服务奔溃
- 存储服务奔溃
# 优势
- 存储计算分离：构建跨多中心、自我管理、容错、自我恢复、独立的存储服务，与上层database service故障分离
- database service仅需要写入redo log到storage service，大幅减少网络IO延迟，增加系统性能
- 故障隔离：隔离因check point等后台服务引起的性能抖动，增强database service的性能稳定性
- backup/restore：TBC