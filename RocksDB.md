---
title: RocksDB
tags: 
renderNumberedHeading: true
grammar_cjkRuby: true
---
# 前言
以下是关于rocksdb的一些事实：
- fork自level db
- 在level db的基础上增加了一些feature
- 由facebook开发
- 开发语言是C++
- 继承了level db的memtable , sst等机制

# 架构模块
### 架构层次
##### 写数据
![写数据-图片来自网络](https://gitee.com/string_coder/xiaoshujiang/raw/master/write.jpg)

如上图，写入时
- 元数据会被追加到memtable
 	 - memtable位于内存区域。
	 - 由此可见，对于写线程来说，写入内存即可返回，大大提高了写的速度
	 - 内存写满了以后，打包放入“待持久化pipeline”
- 操作会被写入WAL文件。因此，个人觉得，WAL相当于操作日志。通过这个日志，应该可以回放数据，比如故障后恢复memtable中丢失的数据。
- immutable_memtable持久化的过程，是一个执行LSM算法的过程，同时也是数据按key排序的过程。
- 上面的写数据图，还应该补充一个cache层。【ps：关于cache，写入和更新的算法需要研究的】

##### 读数据
![读数据-图片来自网络](https://gitee.com/string_coder/xiaoshujiang/raw/master/read.jpg)

在读数据时，
- 可以从db读，也可以从snapshot读
- 可能的情况下，会从cache读
- 当cache没有命中的时候，从持久存储读取数据

# 重点特性
![功能点-图片来自网络](https://gitee.com/string_coder/xiaoshujiang/raw/master/functions.png)

根据上图，我们重点从以下几个方面进行说明
### block cache
Block cache是rocksdb用来在内存中存放cache数据的地方，以优化数据读取。
- 用户能传入一个cache object给一个RocksDB 实例。
- 同一进程内多个RocksDB实例能共享同一个cache object。
- 第一级cache object存储的是未压缩的数据
- 还可以设置第二级cache object，存储的是压缩后的数据
- cache 内部都是支持shard，目的是防止锁竞争

具体分为两种类型的Cache:
- LRU Cache
	- 每个shard拥有自己的LRU list，hash查找表
	- 查找与插入都需要先取得对应shard的mutex。查找需要mutex是因为此时LRU list相应的也需要更新

- Clock Cache
	- 每个shard维护一个环形队列
	- cache使用Clock算法来进行Page replacement
	- 此算法的lock粒度比LRU Cache要好，因为在查找时，无需更新环形队列，故查找无需在shard进行加锁

### SST File
- SST文件是一段排序好的表文件，它是实际持久化的数据文件。里面的数据按照key进行排序能方便对其进行二分查找。在SST文件内，还额外包含以下特殊信息：
	- Bloom Filter，用于快速判断目标查询key是否存在于当前SST文件内。
	- Index / Partition Index，SST内部数据块索引文件快速找到数据块的位置。

### Iteration Operation
- RocksDB能够支持区间范围内的key迭代器的遍历查找。

### Ingesting SST files
- 当用户想要快速导入大批量数据到系统内时，可以通过线下创建有效格式的SST文件并导入的方式，而非使用API进行KV值的单独PUT操作。

### Delete Range
- 区间范围的删除操作，比一个个Key的单独删除调用使用更方便。

### Transaction
- RocksDB内部提供乐观式的OptimisticTransactionDB和悲观式(事务锁方式)的TransactionDB来支持并发的键值更新操作。

###### TransactionDB
- 使用TransactionDB时，为了进行冲突检测，所有相关key都会被加锁。
- 在具体实现上
	- 悲观事务实现了readrepeatable级别的隔离性。
	- 默认模式下，Put操作会对相关Key加锁，GetForUpdate同样。
	- 默认模式下，采用MVCC方法，利用log sequence number(LSN)和snapshot实现。
	- 在2PC模式下，prepare阶段会对所有相关key进行加锁

###### OptimisticTransactionDB
- 乐观事务在prepare write阶段，不会持有任何锁；她在commit 阶段进行冲突检测，以便确定没有其他writers修改她正在操作的keys
- 具体乐观事务是怎么进行冲突检测的？是不是也是通过对所有key加锁的方式？这个还有待探讨

###### 乐观事务与悲观事务的比较
 根据上述乐观事务与悲观事务的比较，在并发写场景下，我们可以分析得出：
- 如果冲突比较多，则悲观事务比较合适，但悲观事务的资源消耗较大；原因是其对所有key进行加锁。
- 若冲突少，以偶发冲突为主，则乐观事务的性能比较高，资源消耗少；比如一个场景：non-transaction很多，有一些transaction。

### Snapshot
###### 概念
Snapshot（快照）是关于指定数据集合的一个完全可用拷贝，该拷贝包括相应数据在某个时间点（拷贝开始的时间点）的映像

- 原理
 	- rocksdb在插入每条记录的时候会包含一个log sequence number(LSN)，rocksdb使用该sequence number作为时间标志来区分不同的snapshot
 	- 对于整个key-value存储状态，Snapshot提供了一致性只读视图。

 
- 有什么好处
	- 能够进行在线数据备份与恢复
	- 在事务处理时，作为数据副本，实现MVCC方法

###### 使用
- Snapshots通过方法DB::GetSnapshot()创建
- 当snapshot不再使用时，需要用DB::RealeaseSnapshot释放
- ReadOptions::snapshot不为NULL时表示读取数据库的某一个特定版本。如果Snapshot为NULL，则读取数据库的当前版本。

# 读写过程

