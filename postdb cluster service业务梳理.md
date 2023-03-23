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
storage node在发送vote response时，vote和不vote都会回复，使得pcs node不用等到election timeout才进入下一轮，加快选主效率

异常处理：
storage node保存vote for信息，避免voter在同一个term投2次票

## 将选主结果通知给本Node其他模块


## 将选主结果通知给其他node



# pcs node数据的持久化

# pcs node间数据的同步

# 选择slice leader
## 策略
##  哪些node需要知道slice leader信息
## 这些node怎么知道某个slice的leader

# node下线状态判断
## 利用租期机制
1. 原理上周说过
2. 与etcd不同的是，postdb


# node下线后的调度
1. node类型： computer node , slice leader
	1. 重新选择slice leader
	2. 通知新slice leader：原 leader已经下线，需要
	
2. node 类型:  computer node , slice replica 
	通知 slice leader：replica 已下线
	
3. node 类型:  storage node
	？
