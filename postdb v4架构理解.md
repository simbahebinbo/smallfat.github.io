---
title: postdb v4 架构理解
tags: 
grammar_cjkRuby: true
---
# 需求
## V3版本中存在的问题

## V4的目标


# 本地化架构、理解、问题

![enter description here](./images/Screenshot_from_2022-11-28_15-34-37.png)

## 对流程的初步理解（暂时不考虑recovery情况）
### 初始化流程
- 网络联通，节点发现
- pcs模块启动并选主，从存储层载入meta-data

### 写流程（DML）
- driver将client write request转发到对应shard primary node (疑问：driver如何知道哪个是对应的shard primary node)。
- shard primary node上，computer layer解析并执行plan，数据写进page buffer，且生成了对应的WAL。
- 若发过来的request没有对应的shard，pcs创建一个新的shard，并选举某个node作为新shard的primary node(leader)，合并入对应的shard group。pcs将shard信息写入相应的meta data。
- WAL日志将被复制到该shard的所有副本node，使用quorom协议决定wal是否复制成功。
- primary node及相应的副本上，WAL被持久化到storage layer；同时page data也被持久化(如何持久化存储需另外考虑)。

### 读流程
- driver将client read request转发到某个node。
- node 解析该request
- 依据request中不同shard的数据，分别到shard所在的primary node上读取数据。shard对应的primary node号，从pcs查询可知。

## 扩容/缩容
- 新增node时，pcs根据负载均衡原则调整shard分布，从负载重的node reload一些shard-group到新的node，并更新meta data
- 减少node时，从meta data中取得该node的shard/shard group/wal信息，pcs根据负载均衡原则调整shard分布，并在目标node中replay相关wal


## pcs的功能
- 进行pcs leader election，确定pcs primary node。
- 创建shard和shard group，记录相关信息到meta data中，并作持久化
- 进行shard leader election，确定shard的leader。
- 扩容/缩容
	- 扩容新增node时，pcs primary node根据负载均衡原则调整shard分布，从负载重的node reload一些shard-group到新的node，并更新meta data
	- 缩容减少node时，从meta data中取得该node的shard/shard group/wal信息，pcs primary node根据负载均衡原则调整shard分布，并在目标node中replay相关wal
- 提供查询接口，查询shard信息：shard_key_range/primary_node/replications 信息
- 全局配置/统一监控/日志/Tracing
	- 提供API接口
	- 各节点有agent，pcs primary 向各节点agent发出命令，获取或设置各节点数据

## 细节推测
1. 每个node上wal与shard的对应关系?
	- 在compute layer，wal将被按照relation/page分解为一段段的wal，shard可以对应wal lsn为key。这样，wal与shard是一一对应的关系。
2. shard什么被创建？ 
	- 在wal被compute layer创建之后
3. shard的key
	- 同第一点分析
4. shard key-range生成策略
	- 范围
		- 因lsn的最新值是不断增长的，怎么处理？
			- 使用shard group容纳新的shard
		- 要考虑负载均衡问题
		- 要考虑范围大小
	- 时机 
		- shard创建之时
5. shard元信息保存在pcs内存，并持久化到disk
6. 全局配置信息存在哪里？
	- 保存在每个node的存储层，作为元信息的一部分
7. meta-data具体包含几种类型
	- shard信息 ： key range/primary node/replication nodes
	- 配置信息
	- table/index等元数据
	

## question
1. 写DDL时，表的结构会发生变化，其元数据为什么要存在meta data内，而不是与DML相同的处理方式？
	- 元数据指的是表的结构，比如table/index等的字段属性等。数据是构建在元数据之上的。
	- 在存储层，表结构的元数据是单独存储的。	
2. meta-data在各node间，需要保证一致性吗?

