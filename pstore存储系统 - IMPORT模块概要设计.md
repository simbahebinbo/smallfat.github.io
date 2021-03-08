---
title: pstore feature - bulk data import details design
tags: pstore IMPORT
renderNumberedHeading: true
grammar_cjkRuby: true
---
# 背景
- pstore是一个K-V存储系统
- IMPORT模块是pstore系统的批量数据导入模块


# 目标
本模块旨在建立将大规模数据导入pstore的接口和通道，总体目标分为功能与性能两个维度：
### 功能
- 在libpstore层提供import接口，并支持块数据的提交
- 导入的数据要满足节点上数据基本均衡的要求


### 性能
- 支持大规模数据持续长时间导入
- 导入速度要比put操作有数量级的提升


# 系统设计
### 系统视图

![系统视图](./attachments/1612839742194.drawio.html)

- pstore cluster里面有3个node，每个node上有N个tablet。tablet有leader/follower角色
- client是指调用了libpstore库的应用
- client通过调用node server提供的import service api接口导入数据。该接口以RPC形式提供。

### 功能层次

![功能层次图](./attachments/1612860096787.drawio.html)

### 主要功能
###### libpstore 
import api是libpstore提供的外部接口。主要的功能点有：
- Data Sort
	- 在给rocksdb ingest数据的时候，一个已按key值排好序的sst文件有利于提升其LSM树的compaction效率
- Data Dispatch
	- 在进行数据导入的时候，我们需要把数据近似均匀地分布在各个tablet上
	- 把相应key值范围的数据dispatch到对应的本地缓存中
	- 由Data Send功能将上述缓存中的数据发到对应的tablet leader
- Import Task Manager
	- 客户端进行数据导入的基本单位是一个导入任务
	- 每个导入任务有一个任务ID
	- 导入任务有start/importing/imported/failed四种状态
	- 导入任务管理则是对导入任务和这些状态进行管理
- Data Send
	- 本地缓存中的数据需要发送给对应的tablet leader。libps中提供了找到对应tablet leader的node id的方法。
	- 调用对应node上pstore service层的RPC接口，就可以把数据发送过去

###### service 
service层是pstore server端的服务层，这些service都以protobuf作描述，以RPC接口的方式提供。
- pstore service::Import Api
- file-copy service::Data Fetch Api

###### tablet replica
每个node上存在多个tablet。tablet replica需要处理如下功能点：
- SST Generate
    - 通过调用RocksDB的接口，可以生成独立于rocksdb系统的外部sst文件
- SST Ingest
	- 通过调用RocksDB提供的接口，可以将外部sst文件导入进LSM树，成为db中的数据
- ImportTransaction
	- Import Transaction的功能是处理Import任务的数据一致性，包括raft log replication，.sst文件导入进数据库等
- - Transaction Driver
	- 此driver用来对raft协议下的replication过程进行控制，从而完成tablet leader和follower之间的数据同步，包括raft log和sst文件同步
	- 在具体实现层次，需要修改原有Transaction Driver的驱动机制，Import Transaction也需要作为驱动机制中的一个实体
	- Import Transaction处理中，当raft log完成了replication，follower需要从leader同步.sst文件。同步.sst文件完成后，进行.sst文件的即Ingest工作（即Apply）

### 主要功能流程
###### 序列图

![绘图](./attachments/1614304810177.drawio.html)

### 异常处理
- 异常返回
- 异常定位
   
# 影响和限制
- 不支持对整个IMPORT操作的事务
- 导入过程中，不能对tablet的key范围作调整 