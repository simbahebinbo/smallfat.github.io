---
title: PostDB - WAL日志空洞管理模块设计
tags: WAL 
category: /小书匠/日记/2021-07
renderNumberedHeading: true
grammar_cjkRuby: true
---


# 功能
### 空洞发现
- 存储区间列表
	- 此列表记录了所有已进行持久化的数据的区间
	- 区间以lsn标示，包括上标和下标
	- 上标和下标都带有原始record所指向的前一个record的lsn信息
- 空洞发现算法

![绘图](./attachments/1626064228154.drawio.html)


- Q： 
	- 没有gossip/不考虑CRC校验
	- 间隙数据的持久化（??）
	- 间隙补充的时候，调用接口
	- wal并不一定单调递增

### 空洞同步

![绘图](./attachments/1626068651977.drawio.html)



### 异常处理
###### 主机宕机


###### 网络故障
- 发生网络故障时，比如网络中断/网络超时等，

# 协程
- 日志空洞同步模块的主循环运行在独立的协程内；
- 日志间隙发现 - 并行进行Record CRC校验，须同时开启多个协程进行计算；
- gossip通信协议运行在独立的协程内





