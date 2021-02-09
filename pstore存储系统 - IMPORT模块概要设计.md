---
title: pstore存储系统 - Feature IMPORT 概要设计
tags: pstore IMPORT
renderNumberedHeading: true
grammar_cjkRuby: true
---
# 背景
- pstore是一个K-V存储系统
- IMPORT模块是pstore系统的数据导入模块
# 目标
本模块旨在建立将大规模数据导入pstore的接口和通道，总体目标分为功能与性能两个维度：
### 功能
- 在pstore层提供标准api接口，对具体客户端类型解耦，支持块数据的提交
- 导入的数据要满足节点上数据基本均衡的要求
### 性能
- 支持大规模数据持续长时间导入
- 导入速度要比put操作有数量级的提升
# 基本架构

![基本架构](./attachments/1612839742194.drawio.html)

- 如图，cluster里面有3个node，每个node上有N个tablet。tablet有leader/follower角色
- client通过调用node server提供的api接口导入数据。该接口以RPC形式提供。

# 系统设计
### 框图
### API 接口设计
### 数据导入
###### 在tablet上进行导入
###### 生成sst文件
###### 导入sst文件进db
###### 导入数据的同步机制


### 异常处理

# 对其他系统的影响