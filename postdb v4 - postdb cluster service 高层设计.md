---
title: postdb v4 - postdb cluster service 设想
grammar_cjkRuby: true
---
# 背景与目标


# 在postdb-v4系统中的位置

![enter description here](./images/Screenshot_from_2022-12-15_09-48-31.png)



# 状态图

## pcs node 状态

![绘图](./attachments/1670310960410.drawio.svg)

- pcs_election - 选举 primary pcs
- pcs_recovery - pcs数据recovery
- pcs_running - 正式提供pcs 服务，此状态是pcs node的稳定态

# 功能
## pcs group

![PCG Group](./attachments/1671089515758.drawio.svg)
- 整个cluster中，pcs 用来控制整个cluster的状态变化。使用pcs group来保证pcs node的可用性
- pcs group之间采用与Raft类似的协议，选举primary pcs节点，并在pcs之间同步元信息


### pcs 选举

- pcs 节点在进行选举时，状态迁移如下图所示：

![enter description here](./images/Screenshot_from_2022-12-07_09-43-40.png)
- 节点选举是在pcs 节点之间进行的，pcs 节点是candidate 与 follower


## pcs间元信息同步

![元信息同步](./attachments/1671107410817.drawio.svg)
- pcs node之间数据同步是通过同步“PCS WAL”到replica pcs 节点完成的
	1. primary pcs节点接收到write请求，写入MetaData Buffer
	2. primary pcs节点生成PCS WAL
	3. PCS WAL同步到replica pcs节点
	4. replica pcs节点收到PCS WAL后，进行持久化并replay WAL，数据写入Buffer	

## pcs元信息
### 元信息类别
- shard
	- shard id
	- key range of shard
	- primary node
	- replica nodes
- shard-group
- 配置信息
- system table


### 元信息的读写
- 写(只能在primary pcs上)
	- create table(index...) - 创建分片
	- drop table(index...) - 回收分片
	- 平移分片
	- 分裂分片
	- 合并分片

- 读(任一个pcs上)
	- select
	- insert/update
	- ...

## shard/shard group管理

### 创建shard
#### 分片策略
- key range公式模板 - 1. 按容量切割  2. [start -end) 
- 位置分布(primary shard/replica shard)
	- 指定机器
	- 负载均衡
	- 考虑地理位置
- 副本数(全局参数)
- shard-group策略
	- 用户可设置，优化系统效率

#### 逻辑
- 在primary pcs上执行创建逻辑
- 根据分片策略，计算shard type的 key range，此range为一个左闭右开的区间
- 根据分片策略，计算primary shard 所在的node 
- 根据分片策略，计算replica shards 所在的nodes
- 每个shard实例(包括primary shard/replica shards)都是同一个shard-id
- 将上述信息写入metadata，并同步到replica pcs/learner
- primary pcs 通知指定node为primary shard

### 回收shard
#### 逻辑
- 在primary pcs上执行回收逻辑
- 从metadata中查询目标分片的所有实例位置
- 调用存储层接口，在指定节点回收shard实例
- 清除metadata中对应shard信息
- 同步到relica pcs中

### 平移/分裂/合并
- waiting

## 扩容/缩容
- waiting

## cluster node状态管理

- primary pcs利用心跳机制定时收集cluster内各node的状态，包括：
	- 在线情况
	- 其他业务指标(shard数，shard总占用资源，io负载，cpu负载，memory占用，硬盘空闲容量等)
- 此状态信息保存在内存cache中，无需持久化，无需被复制到replica pcs: 切换primary pcs场景下，新的primary pcs会重新收集最新cluster node最新状态信息


## 控制命令下发
### 下发机制
### 命令类型
- 配置（WAL）
	- 整体配置
	- 单项配置
- 日志命令
- tracing
- 监控

# question
1. Online DDL：
元数据的变更，由PCS 写入，每个表有自己的元数据多版本;
