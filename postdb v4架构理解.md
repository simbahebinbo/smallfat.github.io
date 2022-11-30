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
- 网络联通，节点注册，节点发现
- pcs模块启动，从存储层载入meta-data，给存储层下发全局配置

### 写流程（没考虑DDL）
- driver将client write request转发到对应shard primary node。
- 若转发过来的request没有对应的shard，pcs创建一个新的shard，并选举某个node作为新shard的leader，合并入合适的shard group。最后，pcs将shard信息写入相应的meta data。
- primary node上，computer layer解析并执行plan，数据写进page buffer，且生成了WAL。
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
- 进行shard leader election，确定shard的leader
- 在shard和shard group被创建完成后，记录相关信息到meta data中，并作持久化
- 响应node状态变化(新增/减少/扩容/缩容)
	- 新增node时，pcs根据负载均衡原则调整shard分布，从负载重的node reload一些shard-group到新的node，并更新meta data
	- node减少时，从meta data中取得该node的shard/shard group/wal信息，pcs根据负载均衡原则调整shard分布，并在目标node中replay相关wal
- 提供查询接口，查询shard信息：shard_key_range/primary_node/replications 信息
- 全局配置： 


## question
0. 每个node上wal与shard的对应关系?
0. shard什么被创建？ - 在wal被创建之后？
1. shard的key：wal lsn？
3. shard key-range生成策略
	- 范围
		- 因lsn的最新值是不断增长的，怎么处理？
		- 要考虑负载均衡问题
		- 要考虑范围大小，这关系到
	- 时机 - shard创建之时
2. 响应node变化(增加/减少)
	- 引起sharding key range的变化 - 要及时推送到存储层，重构存储
	
3. sharding元信息保存在pcs内存，并持久化到disk
4. 全局配置信息存在哪里？
5. DDL: 为什么要存在meta data内？

