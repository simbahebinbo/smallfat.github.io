---
title: postdb cluster service业务梳理
tags: 
grammar_cjkRuby: true
---
# 需求
1. pcs node之间组成一个高可用集群，通过选举产生leader，pcs leader负责所有pcs逻辑 
2. pcs node本身应该是无状态的 
3. pcs负责选择slice leader
4. pcs负责所有node的下线状态判断
5. 作出node 下线判断后，进行相应调度

# pcs选主
## 选主流程
pcs node 发起 request vote 给 storage node
storage node 在每个term只能给一个pcs node投票
每一轮投票里，得到多数派投票的pcs node被选为leader
当选为 pcs leader的node给其他 pcs node发送心跳
接收到更大term 消息的pcs node，自动转为follower

优化点：
pcs node在给voter 发送request vote时，等到至少多数voter都是Online连接状态时发送
pcs node 在转到新的term时，持久化存储保留该term，重启时load term

异常处理：
storage node保存vote for信息，避免voter在同一个term投2次票

**工作进度：代码已完成**

# pcs 管理哪些数据
slice leader信息
node status信息，如下线状态，cpu使用率， 硬盘使用信息等
为维持node status所采取的相应机制（比如lease机制）的相关数据，如每个node的租约信息

# pcs 数据持久化和数据通知机制

pcs数据的特点：少量，关键数据，更新频率不高
pcs需求：无状态节点

**方案0：**
可以采用类似Postdbv3中计算节点数据持久化机制，pcs的数据存在在storage节点，需要处理数据的不一致问题
因pcs数据量较少，此方案对于pcs来说有点重

**方案1：** 

![绘图](./attachments/1680233830749.drawio.svg)
1. client(pcs其他模块)向PCS Leader请求写入数据，pcs leader将数据存入内存，
2. pcs leader 向follower发出请求，follower回复同意(不同步数据)
3. leader判断回复的数量满足quorum多数派要求后，写入到meta postdb中。
4. 写入完成后，pcs leader回复client写入完成
5. pcs leader把数据推送到其他计算节点的PCS front buf（此数据带有pcs term 和 term version），其他计算节点可直接从该组件读取pcs数据。pcs的最新term和最新term version会随着心跳吧发送到计算节点。计算节点从PCS front buf读取数据时，pcs front buf会比较term和term version，判断数据版本。
6. PCS follower向meta postdb订购PCS日志，同步pcs数据

meta中pcs相关数据是经过多数派同意的，是可以信任的
pcs follower从meta 订阅日志流并回放来同步pcs数据
其他node通过从PCS front buf获取pcs 数据

网络分区场景：
因为写入meta中的pcs数据是需要满足多数派的，所以此情况下多leader的异常可以避免。

node crash场景：
	pcs leader crash:
		client发出write request，在数据由pcs leader写入meta 之前，pcs leader crash后，client判断此次写入超时失败
		pcs leader crash会引起pcs election，重新选择新的leader
	
	pcs follower crash:
		follower crash之后重启，从meta复制数据，并订购meta 日志流，进行数据恢复

优点：
pcs节点无状态，可随时在其他node上启动pcs实例
pcs数据通过meta进行持久化
其他node可通过pcs front buf获取pcs信息，减少了网络流量，避免网络拥挤，且不从follower读取数据，使得follower同步数据可以延后，提高写入效率

缺点：
pcs 对meta有依赖 ，meta node(计算及存储节点) crash了pcs也无法工作 
因此, meta crash了，所有node都将受到影响，因此首先应该将meta node 恢复正常


# 选择slice leader
## 策略
根据node的负载，选择负载最小的node成为它所在slice的leader

##  哪些node需要知道slice leader信息
所有业务 node

## 这些node怎么知道某个slice的leader
可以由pcs leader推送消息

===============================================
# node下线状态判断
## 利用租期机制
1. 原理上周说过


# node下线后的调度
1. node类型： computer node , slice leader
	1. 重新选择slice leader
	2. 通知新slice leader：原 leader已经下线，需要
	
2. node 类型:  computer node , slice replica 
	通知 slice leader：replica 已下线
	
3. node 类型:  storage node
	？

==================
**方案2：**
此方案下PCS为完全无状态的模块

leader切换： 选主成功后，新leader向所有业务node发出数据更新请求，业务node向pcs报告自己的数据，据此，新leader重获所有最新node相关数据(slice leader/lease相关信息/)

网络分区场景：
	出现双leader/脑裂现象：
		新的leader在old_term + 1会重新从node收集数据
		
	维持原先leader：
		不影响

node crash场景：
	pcs leader crash:
		client发出write request，pcs leader crash后，client判断此次写入超时失败
		pcs leader crash会引起pcs election，重新选择新的leader。新的leader会强制更新pcs node 数据
	
	pcs follower crash:
		follower crash之后重启，从leader复制数据，进行数据恢复

优点：
完全无状态
所有PCS节点crash以后，可重启老节点或在新的节点上启动pcs

缺点：
leader切换后，PCS中只有即时node数据