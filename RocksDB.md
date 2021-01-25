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
	- 此算法的lock粒度比LRU Cache要好，因为在查找时，无需更新环形队列

### SST File
- SST文件是一段排序好的表文件，它是实际持久化的数据文件。里面的数据按照key进行排序能方便对其进行二分查找。在SST文件内，还额外包含以下特殊信息：
	- Bloom Fileter，用于快速判断目标查询key是否存在于当前SST文件内。
	- Index / Partition Index，SST内部数据块索引文件快速找到数据块的位置。

### Iteration Operation


