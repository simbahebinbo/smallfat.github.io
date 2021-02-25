---
title: pstore系统 - Feature： IMPORT bulk data 设计文档
tags: pstore IMPORT
renderNumberedHeading: true
grammar_cjkRuby: true
---
# 背景
- pstore是一个K-V存储系统
- IMPORT模块是pstore系统的数据导入模块
# 目标
本模块旨在建立将大规模数据导入pstore的接口和通道，总体目标分为功能与性能两个维度：
### 功能
- 在pstore层提供标准api接口，对具体客户端类型解耦，支持块数据的提交
- 导入的数据要满足节点上数据基本均衡的要求
### 性能
- 支持大规模数据持续长时间导入
- 导入速度要比put操作有数量级的提升


# 系统设计
### 系统视图

![系统视图](./attachments/1612839742194.drawio.html)

- 如图，cluster里面有3个node，每个node上有N个tablet。tablet有leader/follower角色
- client通过调用node server提供的api接口导入数据。该接口以RPC形式提供。

### 功能层次

![功能层次图](./attachments/1612860096787.drawio.html)

### 接口设计

![绘图](./attachments/1612929222347.drawio.html)

### 主要功能
###### 数据的均衡导入
- 在进行数据导入的时候，我们需要把数据平均分布在各个tablet上
- 通过获取各个tablet的key的范围，把原始数据中对应的key范围的数据，发送到对应的tablet，从而实现数据的均衡导入
- 如何找到对应的tablet - 数据需要发送给对应的leader tablet。libps中提供了找到对应tablet leader的方法

###### 大规模数据的持续导入
- 由于IMPORT Feature主要应对数据规模较大的场景，传统的rpc方法调用模式效率不高，因此应该采用数据传输效率更高的rpc流模式来处理Client与Server之间的数据传输。

###### 生成sst文件
- 通过调用RocksDB提供的接口，可以生成独立于rocksdb系统的外部sst文件

###### 导入sst文件进db
- 通过调用RocksDB提供的接口，可以将外部sst文件导入进LSM树，成为db中的数据
  
###### 导入数据的同步机制
- 与普通put不同，IMPORT的原理是直接injest文件进LSM树，因此数据的同步（指raft协议下leader向follower同步数据）机制也不一样
	- log的command不是具体数据，而是sst文件位置的描述
	- follower复制此log以后，在状态机执行此command时，会去leader节点上拉取对应的sst文件，并执行injest操作
	- follower在上述command执行成功后，再返回成功

### 主要流程

### 异常处理
 ### 问题
 - sst文件injest是否支持事务
 - 如果不支持，是否存在数据状态未知问题
   
# 影响和限制
- 不支持对整个IMPORT操作的事务
- 导入过程中，不能对tablet的key范围作调整 