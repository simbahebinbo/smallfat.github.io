---
title: pstore存储系统 - IMPORT模块概要设计
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
- 对健壮性有较高的要求，支持断点续导
- 支持多任务导入
- 导入的数据要满足节点上数据基本均衡的要求
### 性能
- 支持大规模数据持续长时间导入
- 导入速度要比put操作有数量级的提升
# 基本架构



# 系统设计


# 对其他系统的影响