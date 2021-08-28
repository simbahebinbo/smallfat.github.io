---
title: PostDB - wal日志空洞管理模块详细设计
tags: WAL 
category: /小书匠/日记/2021-08
renderNumberedHeading: true
grammar_cjkRuby: true
---


# 功能
### 总体功能

![绘图](./attachments/1626283921116.drawio.html)

###### 模式
本模块在不同的场景下，分别工作于两种模式下：普通模式/recovery模式
- 普通模式
	- 正常工作模式下，根据PGCL进行空洞填充
	- 正常模式下，主模块调用本模块时，会传递本次已持久化的lsn区间[first_lsn, last_lsn)和PGCL信息，本模块负责将其合并到存储状态列表
- Recovery模式
	- Recovery模式下，本模块负责对不同阶段，进行空洞填充，并发送补齐点
	- 分为尝试日志补齐和最终日志补齐两个阶段。
		- 尝试日志补齐阶段，recovery模块会传递"consistency_term/[consistency term的最小起始位置, 最大NWL]"信息
		- 最终日志补齐阶段，recovery模块会传递"最终日志补齐点“信息
###### 总体功能流图

![绘图](./attachments/1629356917623.drawio.svg)


### 存储区间合并
###### 存储状态列表
- 列表记录了所有已完成持久化的数据的索引状态
- 每个列表item包含如下信息：lsn, range_type, sent_hole
	- lsn：连续区域里第一个lsn
	- range_type：区间内是空洞/非空洞/结束节点
	- sent_hole：空洞区间是否已发送给peer。此字段只在hole区间有效
- 区间的表示：[item1.lsn, item2.lsn)，初始区间[INIT_VALUE, max(uint64)), INIT_VALUE默认值为0

- 在系统初始化时，创建存储状态列表
- 在系统正常退出时，保存存储状态列表至硬盘系统配置
- 在storage node启动时，恢复存储状态列表。具体恢复流程见异常处理


###### 存储区间合并

- 算法

![绘图](./attachments/1627628363001.drawio.html)

- 已有空洞与新空洞
	- 在持续的空洞扫描过程中，原有空洞可能会出现部分被填充，继而分裂为两个空洞。在向其他peer寻求数据的时候，要正确处理hole_sent标志 - 分裂的空洞将不继承原空洞hole_sent标志，重新变为未发送状态


### 空洞填充
#### 请求状态列表
- req_id
- peer_id
- req_time
- head_peer_id

#### peer存储状态列表
- peer_id
- peer_storage_status - 各peer的lsn存储状态，可能不是最新数据。在recovery模式进入时，一并获取
- term_range - 各peer的term range，其中term id 为本storage node term id。
	- term
	- begin_lsn
	- end_lsn

#### 普通模式下的空洞填充  
###### 空洞发现与空洞数据请求
- 流程

![绘图](./attachments/1627873970845.drawio.html)
- 数据寻址
	- 为做到peer负载均衡，每个gap第一次进行空洞数据请求，以round robin形式，选择满足特定条件的peer node为目标节点。条件： 
		- peer_PGCL >= PGCL
		- 从上一个gap的目标peer为起点开始round robin（不包括此peer node)
- 获取数据
	- 向上述被选中的peer发送获取空洞数据请求(参数：空洞区间)后，更新请求状态列表
- 不要重复请求同一空洞区间
	- 所有已经发送到peer的空洞数据请求，hole_sent标记为“已发送”状态
- req_id的生成算法：
	- 随机hash? 
	- lsn区间与时间组合？	
###### 空洞数据填充

![绘图](./attachments/1627887572129.drawio.html)

- 填充空洞数据
	- 调用既有接口，将空洞数据填充到日志持久化存储中
	- 填充之前先判断存储区块列表中是否包含相应区间，包含则说明已填充完毕，无需再填充


###### 请求状态检测与处理

![绘图](./attachments/1629432112010.drawio.svg)

- 主要处理以下场景
	- 某个请求超时 - 继续向下一个peer请求数据

#### recovery模式下的空洞填充
###### 空洞发现与空洞数据请求
- 尝试补齐阶段：流程

![绘图](./attachments/1629428465960.drawio.svg)

- 最终补齐阶段：流程

![绘图](./attachments/1629686639906.drawio.svg)

- 数据寻址算法
  - 取得所有peer node的最新存储状态列表，存入对应的peer_storage_status
	- 为做到peer负载均衡，每个gap第一次进行空洞数据请求，以round robin形式，选择满足特定条件的peer node为目标节点。条件：
		- peer_storage_status包含或者部分包含当前gap区间
		- 从上一个gap的目标peer为起点开始round robin（不包括此peer node)
- 获取数据
	- 向上述被选中的peer发送获取空洞数据请求(参数：空洞区间)后，更新请求状态列表
- 不要重复请求同一空洞区间
	- 所有已经发送到peer的空洞数据请求，hole_sent标记为“已发送”状态

###### 空洞数据填充

![绘图](./attachments/1627887572129.drawio.html)

- 填充空洞数据
	- 调用既有接口，将空洞数据填充到日志持久化存储中
	- 填充之前先判断存储区块列表中是否包含相应区间，包含则说明已填充完毕，无需再填充

###### 请求状态检查及处理

![绘图](./attachments/1629682848099.drawio.svg)

- 主要处理以下几个场景
	- 某个请求超时
	  - 某个请求超时 - 继续向下一个peer请求数据
	  - 某个请求已经向所有peer都进行了请求，且都超时 - 清除本请求，不再发起请求
	- 所有请求都处理完毕 - 发送当前补齐点（即NCL）


### 异常处理
###### 主机宕机
主机宕机有可能造成存储区间列表不能正常退出序列化，从而影响重启后正常的功能。因此需要重新构建存储区间列表
- 流程

![绘图](./attachments/1626285222578.drawio.html)

- 从持久化日志存储中重构存储状态列表 - 由其他模块完成后，将区间信息传递到本模块，进行重建

###### 网络故障
- 本模块在进行空洞填充时会通过网络发送接受日志数据
- 普通模式下，发生网络故障时，比如网络中断/网络超时等，目前会延迟空洞填充补齐的时间，并不影响系统的功能性和健壮性
- Recovery模式下，网络故障可能会导致某些storage node不能及时到达补齐点，从而无法满足N/2+1的quorum规则，本轮recovery失败

### 接口
###### 与主模块的接口
- 调用
	- 写log数据 - 在空洞填充时调用
	- log数据获取 - 指定page区间，从本地取得指定数据
	- log截断 - 在recovery模式下调用
- 被调用
	- 区间合并
	- 尝试空洞填充
	- 最终空洞填充

###### 与其他模块的接口
- 无

