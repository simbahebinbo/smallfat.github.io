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
- 配置信息
- system table (include partition/table/... 结构元数据)
- shard


### 元信息的读写场景
- 写(只能在primary pcs上)
	- create table(...) - 创建分片
	- drop table(...) - 回收分片
	- 平移分片(主动/被动)
	- 分裂分片(主动/被动)

- 读(任一个node上(包括pcs meta data learner节点))
	- select
	- insert/update
	- client driver

### shard/shard group管理

#### 分片策略
##### 自动/手动分片
- 手动分片
	- 由用户指定分片数、分片主键(primary key)、副本数、分片范围(可选)等
	- 在执行创建relation DDL的时候，根据上述参数，创建shard
	- 分片合并/分裂/平移可以手动进行
	- 缺点是数据分布不均匀。数据分布不均可能导致数据库负载不平衡，从而使其中一些节点过载，而另一些节点访问量较少。

- 自动分片
	- 系统在运行过程中，会根据系统节点负载情况，动态地进行分片合并/分裂/平移等操作，以应对某些节点过载过热等情况
	- 最终目标是形成一个弹性伸缩的分片集群
	
- 问题：从需求和资源来看，先做基础的手动分片？

##### 分片方式
- range
	- 优点
		1. 基于range的分片很容易实现自动分片：只需拆分或合并分片。使用基于哈希的分片的系统实现自动分片代价很高昂
		2. 相对于hash分片，基于range的分片在进行“范围查询”时有优势

	- 缺点
		1. 数据分布不均匀，基于range的分片可能会带来读取和写入热点，可以通过拆分、复制和移动分片消除这些热点。
		
- hash
	- 优点
		1. 数据分布相对均匀
		2. 随即读写单条数据较快
		
	- 缺点
		1. 读写范围数据要跨多个分片，效率差
		2. 分裂分片时，不好处理
		3. 缩容/扩容时，节点变动导致全部分片/数据变动位置。可通过一致性hash方式解决。
	
- 建议使用range
  	- 初期，建议key的选择(key)、分区范围(range)、分区个数都由用户指定或者在全局策略中设置

##### 副本位置
- shard-group
	- 由用户设置
	- 是设置表-表绑定关系？还是shard-shard绑定关系？
	
- 负载均衡策略	
	- 排除哪些节点
		- 硬盘空闲空间 < 某个设置值
		- 当前cpu使用率 > 某个设置值 
		
	- 再考虑shard-group因素：寻找符合shard-group条件的节点
	- 最后使用round robin方法在可用节点上进行选择
		- 可从最简单的策略开始做，后面持续优化

- 灾备策略
	- 主要以存储节点的地理位置为考虑因素，实现灾难备份，如两地三中心，三地五中心等
	- 在存储节点安装时，已经设定地理位置(城市/中心)。此后，系统运行，此信息上报给pcs
	- 给分片副本分配节点时，按中心位置进行分配


#### 创建shard
- 在执行create table(或类似的创建类DDL语句)时，在primary pcs上执行创建逻辑
- 根据分片策略，计算shard 的 key range
- 根据分片策略，计算shard 所在的 nodes(包括primary shard/replica shard) 
- 将上述信息写入primary pcs的metadata，并同步PCS WAL到replica pcs
- replica pcs持久化并回放PCS WAL
- Quorum成功，选择primary shard node
- primary pcs 通知指定node为primary shard以及相关的shard信息，此后关于此shard的事宜由此node负责


#### 回收shard
- 在执行drop table(或类似的创建类DDL语句)时，在primary pcs上执行回收逻辑
- 从metadata中查询目标分片的primary shard node位置
- 发送命令给primary shard node，回收shard实例
- 清除metadata中对应shard信息(标记)
- 同步到relica pcs中

#### 读一致性
- 每个wal都是逻辑上独立的，pcs wal也是同样的


## cluster node状态管理
- primary pcs利用心跳机制定时收集cluster内各node的状态，包括：
	- 在线情况
	- 其他业务指标(地理位置，shard数，shard总占用资源，io负载，cpu负载，memory占用，硬盘空闲容量等)
- 此状态信息保存在内存cache中，无需持久化，无需被复制到replica pcs: 切换primary pcs场景下，新的primary pcs会重新收集最新cluster node最新状态信息


## 控制命令下发
- 日志命令
- tracing
- 监控

# question
1. Online DDL：
元数据的变更，由PCS 写入，每个表有自己的元数据多版本;
