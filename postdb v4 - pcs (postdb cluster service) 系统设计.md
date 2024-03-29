---
title: postdb v4 - pcs (postdb cluster service) 系统设计
tags: 
grammar_cjkRuby: true
---
# 背景与目标
- pcs是postdb数据库下的一个模块
- pcs需要收集postdb cluster中所有node的在线状态(通过网络层)
- 在创建table/index/partition等relation,创建对应的分片(shard)时，pcs需要提供对应的策略接口(副本策略，primary shard node选择策略)
- pcs需要应对失效的cluster node(计算节点/存储节点)，进行failover处理
- pcs需要应对shard 容量膨胀情况，进行自动分裂
- pcs需要应对shard 写过热问题，进行自动分裂/平移
- pcs需要支持shard group配置，用户指定的table 中所有shard 存储位置在一起
- pcs需要支持控制命令(日志命令/tracing命令/监控命令等)下发给指定node

# 在系统总体架构中的位置

![enter description here](./images/postdbv4-arch.png)





# 功能
## 模块划分

PCS组件主要存在存在下述需求点：
- 收集cluster中各node的status信息
- 为创建shard提供相关策略支持
- 主动(被动)调度shard：扩容、缩容、平移、分裂等
- 维持PCS组件在cluster中的高可用性，相对应的，PCS之间组成PCS group
- 下发系统命令到指定node
- 维护系统配置信息，保证整个配置存在中心控制，并可下发到各node

相对应的，PCS由以下模块组成：
- 集群状态 Cluster Status
- 策略控制 Policy Control
- 分片调度 Shard Schedule
- PCS 组控制 PCS Group Control
- 命令中心 Command Center
- 配置中心 Configuration Center

![PCS功能模块](./attachments/1672728973669.drawio.svg)


## 组件形态
- Policy Control 模块要被UserSession调用
- 其他模块运行在Extension内


![绘图](./attachments/1672974652093.drawio.svg)

## 模块
### 集群状态 (cluster status)
#### 业务逻辑
- PCS收集整个集群内所有node的下列状态(暂定，实现时依据其他模块需求增减)
	- 在线(online)情况
	- 地理位置
	- io负载
	- cpu负载
	- memory占用
	- 硬盘空闲容量
	
	
- node在本地收集status信息，通过心跳机制，将status信息报告给pcs所在node的coordinator组件；然后coordinator组件会转发给本模块
- 此信息保存在内存cache中，无需持久化，无需被复制到replica PCS
	- 切换primary pcs场景下，新的primary pcs会重新收集最新cluster node最新状态信息

#### 模块接口

1. 更新node status
```
	fn update_nodes_status(s: &node_status) -> boolean;
```

2.  取得最新node status

```
	fn get_nodes_status() -> node_status;
```


### 策略控制 (policy control)
#### 业务逻辑

##### 序列图

![副本位置选择流程](./attachments/1672890309895.drawio.svg)


##### 策略 - 选择shard的primary shard node
- 策略1： 指定固定节点
	- 思路：
		- 用户在系统设置中指定某个计算节点(node ID)为primary shard node
		- 此策略不考虑任何其他因素
	- 流程：
		- 直接返回设置中指定的计算节点
	
- 策略2：负载均衡
	1. 用户可以在系统设置中设置shard-group，绑定了表-表的亲和性关系
	2. 搜索shard-group关系元数据，找出与当前shard所在的表的shard-group
	3. 计算候选节点集合
		- 若上述shard-group不存在，则候选节点集合C为所有正常节点。
		- 若上述shard-group存在，但group中的分片已分配了primary shard node，且(num of primary-shard-nodes) = (副本配置数)，则直接返回该group中已经分配的primary shard nodes
		- 若上述shard-group存在，且(num of primary-shard-nodes) < (副本配置数)，则候选节点集合C为(所有正常节点集合 - 已分配的primary-shard-nodes)
	4. 从候选节点集合中，选择目标primary shard node
		- 原则：按照计算节点的primary shard node频率进行轮转，每个节点机会平等

##### 策略 - 副本位置
- 用户可以在配置中设置系统使用哪种策略：1. 灾备策略； 2. 负载均衡策略
- 策略1：灾备
	- 思路
		- 主要以存储节点的地理位置为考虑因素，实现灾难备份，如两地三中心，三地五中心等
		- 次要考虑因素为负载均衡
		
	- 流程
		- 在数据库系统安装配置时，在系统配置中设定当前集群的地理拓扑结构：地区 - 中心 - 节点三级信息，并对应到每个节点		
		- 在安装配置时，需要检查设置中副本数与当前地区的匹配条件：副本数 >= 地区数，否则达不到每个地区至少有一个副本，无法保证灾备可靠性
		- 执行策略时，按照地区轮转的顺序和负载均衡的原则，进行副本分配
			1. 列出各地区各中心的所有节点
			2. 排除异常节点，得出正常节点集合N
				- 长时间不在线(连续不在线时间 > 临界值)
				- 长时间CPU占用率居高不下(连续CPU高占用率时长 > 临界值)
				- 硬盘(postdb数据目录所在分区)空闲空间 < 临界值				
			3. 搜索shard-group关系元数据，找出与当前shard所在的表的shard-group
			4. 计算候选节点集合C
				- 若上述shard-group不存在，则候选节点集合C为正常节点集合N。
				- 若上述shard-group存在，但group中的分片已分配了副本节点，且(num of 已分配副本节点) = (副本配置数)
					- 若 已经分配的所有副本节点都在正常节点集合N中， 则返回已经分配的所有副本节点
					- 若 已经分配的所有副本节点中有异常节点， 则需要先把该异常节点上的该shard-group中数据转移到正常节点上，再执行上一步
						- 把该异常节点上的该shard-group中数据转移到正常节点上
							1. 针对primary shard node在该异常节点上的shard(属于shard-group的)，重新选择primary shard node; 
							2. 针对该异常节点上的shard(属于shard-group的)，通知对应的primary shard node，进行副本平移操作; 
							3. 若所有primary shard node回复平移成功，则此操作成功；
							4. 若存在失败的回复，则一直持续进行相应shard副本的平移，直到所有shard平移成功
				- 若上述shard-group存在，且从未由shard分配过副本，即：(num of 已分配副本节点) = 0
					- 若 已经分配的所有副本节点都在正常节点集合N中， 则候选节点集合C为正常节点集合N。
			5. 按照地区轮转的顺序和负载均衡的原则，进行副本分配
				1. 选择一个地区，找到候选节点集合C中属于该地区的所有节点
				2. 按照节点的负载(shard副本数等指标)进行排序，优先选择负载最轻的节点(上一轮已经分配了同一副本的节点除外)
				3. 轮转下一个地区，重复执行第1步，直到所有副本都被分配到节点上

- 策略2：负载均衡
		- 执行策略时，按照负载均衡的原则，进行副本分配
			1. 列出各地区各中心的所有节点
			2. 排除异常节点，得出正常节点集合N
				- 长时间不在线(连续不在线时间 > 临界值)
				- 硬盘(postdb数据目录所在分区)空闲空间 < 临界值				
			3. 搜索shard-group关系元数据，找出与当前shard所在的表的shard-group
			4. 计算候选节点集合C
				- 若上述shard-group不存在，则候选节点集合为正常节点集合N。
				- 若上述shard-group存在，但group中的分片已分配了副本节点，且(num of 已分配副本节点) = (副本配置数)
					- 若 已经分配的所有副本节点都在正常节点集合N中， 则返回已经分配的所有副本节点
					- 若 已经分配的所有副本节点中有异常节点， 则需要先把该异常节点上的该shard-group中数据转移到正常节点上，再执行上一步
						- 把该异常节点上的该shard-group中数据转移到正常节点上
							1. 针对primary shard node在该异常节点上的shard(属于shard-group的)，重新选择primary shard node; 
							2. 针对该异常节点上的shard(属于shard-group的)，通知对应的primary shard node，进行副本平移操作; 
							3. 若所有primary shard node回复平移成功，则此操作成功；
							4. 若存在失败的回复，则一直持续进行相应shard副本的平移，直到所有shard平移成功
				- 若上述shard-group存在，且(num of 已分配副本节点) = 0
					- 若 已经分配的所有副本节点都在正常节点集合N中， 则候选节点集合C为正常节点集合N					
			5. 按照负载均衡的原则，进行副本分配
				1. 列出候选节点集合C中所有节点
				2. 按照节点的负载(shard副本数)进行排序，优先选择负载最轻的节点(上一轮已经分配了同一副本的节点除外)
				3. 重复执行第1步，直到所有副本都被分配到节点上				

#### 模块接口

1. 选择primary shard node
```
	fn compute_primary_shard_node(shade_id: Shade_ID) -> (node_id: Node_ID);
```


2. 选择副本位置
```
	fn compute_replica_shard_nodes(shade_id: Shade_ID) -> (node_ids: Vec<Node_ID>);
```


### 分片调度 (shade schedule)

#### 平移shade副本
##### 业务逻辑
- 一个shade，其信息保存在系统元信息表内。
- shade有可能属于某个datagroup，也有可能是独立的。
- 平移属于某个datagroup内的shade，需要将整个datagroup一起移动。这是个原子操作。
- 如果primary shard node回复 平移datagroup内的某个shade 失败，则应该重选副本位置，再次尝试，直到成功



![绘图](./attachments/1672969880164.drawio.svg)
##### 接口
1. 平移shade
```
	fn move_shade(shade_id: Shade_Id) -> boolean;
```

2. 向primary shard node请求移动分片数据
```
	req: (shade_id, source_node, dest_node)
		 
	
	response: (boolean)
```


#### 分裂
- 当shade容量达到临界值时，需要做分裂操作
- 目前的问题：从架构上看，分裂操作，由哪个组件驱动？ 存储层? PCS? Primary Shade Node?
- 若由PCS驱动，应该要涉及分裂算法。
	- 建议使用一致性hash作为shard分裂的方式。
	- 在hash环上新增一个节点。如下图中t4
	
![enter description here](./images/Screenshot_from_2022-12-23_11-17-48.png)





### PCS组控制 (PCS group control)

#### 基本形态

![绘图](./attachments/1672372540325.drawio.svg)
- 整个cluster中，pcs 用来控制整个cluster的状态变化。使用pcs group来保证pcs node的可用性
- pcs group之间采用与Raft类似的协议，选举primary pcs节点，并在pcs之间同步元信息


#### pcs 选举
- pcs 节点在进行选举时，状态迁移如下图所示：

![enter description here](./images/Screenshot_from_2022-12-07_09-43-40.png)
- 节点选举是在pcs 节点之间进行的，pcs 节点是candidate 与 follower


#### pcs间元信息同步

![绘图](./attachments/1672376861618.drawio.svg)

- pcs node之间数据同步是通过同步“PCS WAL”到replica pcs 节点完成的
	1. primary pcs节点接收到write请求，写入MetaData Buffer
	2. primary pcs节点生成PCS WAL
	3. PCS WAL同步到replica pcs节点
	4. replica pcs节点收到PCS WAL后，进行持久化并replay WAL，数据写入Buffer	

### 命令中心 (command center)

## 元信息管理
### 元信息类别
- 配置信息
- system table (include partition/table/... 结构元数据)
- shard

- 问题：
1. system table是否应该作为pcs的元信息？ pcs是以extension的方式存在，在pcs内进行create table操作，相应的系统表元数据都在pcs进程内存空间内，那计算

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

### partition(table) 与shard的对应关系
- 原生postgresql中可以定义partition分区，指数据以range/list/hash三种方式分配进哪个分区。
```
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
) PARTITION BY RANGE (logdate);

CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01');
	
CREATE TABLE measurement_y2006m03 PARTITION OF measurement
    FOR VALUES FROM ('2006-03-01') TO ('2006-06-01') partition by range(peaktemp);
	
create table measurement_y2006m03_pt PARTITION of measurement_y2006m03  FOR VALUES FROM (1) to (10);	

```

- postdbv4中，shard指的是数据存在哪里。shard管理模块不需要关注复杂的partition分区方式，但是需要知道哪个表有哪些partition(partition name)
- partition与shard的对应关系
	- 创建partition(table)时，shard也相应被创建
		- 初始时，partition与shard在数量上是一一对应的
		- 随着数据的更多写入，shard会自动分裂
	- partition(table)被drop时，shard也相应被回收: v3
	- detach


## shard调度管理
### shard分片策略

#### 分片方式
在分片调度层面，分片方式决定了数据被存入了哪个分片。
- range
		1. 基于range的分片很容易实现自动分片：只需拆分或合并分片。使用基于哈希的分片的系统实现自动分片代价很高昂
		2. 相对于hash分片，基于range的分片在进行“范围查询”时有优势
		3. 数据分布不均匀，基于range的分片可能会带来读取和写入热点，可以通过拆分、复制和移动分片消除这些热点。

- hash
	- hash mod
		1. 数据分布相对均匀
		2. 随即读写单条数据较快
		3. 分裂分片时，不好处理
		4. 缩容/扩容时，节点变动导致全部分片/数据变动位置。
		5. 查询范围数据效率低

	- 一致性hash
		1. 数据分布相对均匀
		2. 随即读写单条数据较快
		3. 能较好兼容分裂/缩容/扩容
		4. 查询范围数据效率低

![enter description here](./images/Screenshot_from_2022-12-23_11-17-11.png)

![enter description here](./images/Screenshot_from_2022-12-23_11-17-18.png)



### 创建shard
*- 在执行create table(或类似的创建类DDL语句)时，在primary pcs上执行创建逻辑
- 将create执行的结果保存进primary pcs meta data*
- 根据分片策略，计算shard 的 数量，range等
- 根据分片策略，计算shard 所在的 nodes（存储节点） ，并选择primary shard node（计算结点）
*- 将上述信息写入primary pcs的metadata，并同步PCS WAL到replica pcs*
*- replica pcs持久化并回放PCS WAL*
- 若Quorum成功
- primary pcs 通知指定node为primary shard node以及*发送给它相关的shard信息(*)，此后关于此shard的事宜由此node负责


### 回收shard
*- 在执行drop table(或类似的创建类DDL语句)时，在primary pcs上执行回收逻辑
- 从metadata中查询目标分片的primary shard node位置
- 发送命令给primary shard node - 回收shard
- primary shard node执行命令成功
- pcs 清除metadata中对应shard信息(标记)
- 同步到relica pcs中*

- 提供一个api，清除自身相应元信息缓存

### 分裂shard
- 建议使用一致性hash作为shard分裂的方式。

- 在hash环上新增一个节点。如下图中t4
![enter description here](./images/Screenshot_from_2022-12-23_11-17-48.png)



### 平移分片
- 将某分片从一个node平移到另一个node
- 移动过程中，不影响当前正在进行的服务（热切换）

### Failover
集群在服务过程中，一个服务节点(计算结点或者存储节点，或者同时)发生了crash。 为了提升cluster的可用性，pcs需要进行failover处理

#### 计算结点crash
- 重新计算原属于该节点的primary shard node
- 需要先从元信息取得primary shard node在该节点上的shard
- 计算新的primary shard node完成后，更新元信息，并通知新节点相关shard元信息

#### 存储节点crash
- 重新计算原副本位置在此节点上的shard的副本位置
- 需要先从元信息取得原副本位置在该节点上的shard
- 计算新的副本位置完成后，更新元信息，并通知primary shard node同步shard数据

## 控制命令下发
- 日志命令
- tracing
- 监控


# 状态迁移

![绘图](./attachments/1672372460895.drawio.svg)
