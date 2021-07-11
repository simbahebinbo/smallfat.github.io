---
title: PostDB - WAL日志间隙同步模块设计
tags: WAL 
category: /小书匠/日记/2021-07
renderNumberedHeading: true
grammar_cjkRuby: true
---


# 功能
### 日志间隙管理
##### record列表
##### 间隙发现

##### 间隙同步
###### 协议
- 方案1 - gossip
- 方案2 - kademlia
###### 实现
- 种子节点
### 日志回放


# 序列图

# 线程

# 异常处理

# 对其他模块的影响


# 接口

# source code
### 间隙判定
- server_wal_write
	- ps_driver_write_wal		
		- pswal_write
			- pswal_page
		- ps_advance_ncl
		- ps_advance_ndl
		- pswal_write_flush

- StartupProcessMain
	- StartupXLOG
		- xlog_redo
			- RecoveryRestartPoint
		- heap_redo
			- PageAddItem

### gossip原理与实现
