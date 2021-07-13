---
title: PostDB - WAL日志间隙同步模块设计
tags: WAL 
category: /小书匠/日记/2021-07
renderNumberedHeading: true
grammar_cjkRuby: true
---


# 功能
### 主体流程

![绘图](./attachments/1626062931440.drawio.html)


### 日志间隙管理
##### 间隙发现

![绘图](./attachments/1626064228154.drawio.html)

- 功能点
	- lsn list生成
		- lsn list会作为间隙管理内的lsn索引。在做日志持久化的时候生成。
	- records CRC完整性验证
	    - 在log records持久化以后，硬盘数据因为各种原因，可能发生损坏而导致回放出错， 故需要依从低到高的lsn顺序，对高于NRL的record进行CRC完整性校验。可并行校验，提高校验速度。
	    - record数据寻址: 依据lsn号，得到record所在的segment文件名和offset。注意record跨segment文件的情况。注意segment文件不存在的情况。
	- 向其他节点询问本node上lsn list是否存在间隙
		- 由于上层写日志采用多数派节点确认，因此存在少数派节点可能由于某种原因没有接受到record数据的情况，造成本节点lsn list有空隙存在。故比较时需要比较多数派节点的lsn list来确认本节点lsn list是否有空隙


##### 间隙同步

![绘图](./attachments/1626068651977.drawio.html)



### 日志回放
- 连续性判定 - 在大于NRL的lsn list中，由低位起，开始计算已经同步完成的lsn，直至遇见第一个没有完成的lsn。这个区间内的lsn就被判定为连续的日志，可以触发回放
- 回放
	- 参考或调用目前已有的回放功能
	- 需要注意此处不考虑checkpoint的情况，只是单纯的回放到data page buffer中

### 通讯协议
##### 协议选型
- 有两种同步方案可以选择，gossip协议和kademlia协议。gossip协议更适用于各node上持有相同的数据，kademlia适用于数据保存在不同的分区上的场景。由于间隙同步主要处理同一份数据的不同的副本的场景，这里选择gossip。

##### 协议设计
- 基础协议
	- 本节主要参考redis的gossip协议
	- redis gossip协议已经实现了node间的状态同步等功能
- 业务相关
	为实现我们的功能，需要在gossip协议的基础上增加几个命令：
	- LSN_LIST_GAP_PULL - 向其他peers查询: lsn list gap
	- LSN_LIST_GAP_PUSH - 若其他peer的lsn list相同则向本node发送“相同“回复；若有更完整的list，则向本node发送它的lsn list
	- WHO_HAS_RECORDS_PULL - 向其他peers查询: 谁有指定的records
	- WHO_HAS_RECORDS_PUSH - 其他peers回复：我有指定的records
	- RECORDS_PULL - 向指定peer请求: 请求获取records数据
	- RECORDS_PUSH - 指定peer push records数据


# 调度设计
- 日志间隙同步模块的主循环运行在独立的协程内；
- 日志间隙发现 - 并行进行Record CRC校验，须同时开启多个协程进行计算；
- gossip通信协议运行在独立的协程内

# 异常处理


# 对其他模块的影响
无

# 接口


