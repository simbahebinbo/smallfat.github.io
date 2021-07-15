---
title: PostDB - WAL日志空洞管理模块设计
tags: WAL 
category: /小书匠/日记/2021-07
renderNumberedHeading: true
grammar_cjkRuby: true
---


# 功能
### 总体功能

![绘图](./attachments/1626283921116.drawio.html)

### 存储区间列表
- 此列表记录了所有已进行持久化的数据的区间
- 区间以lsn标示，包括上标和下标
- 上标和下标都带有原始record所指向的前一个record的lsn信息
- 在系统初始化时，创建存储区间列表
- 在系统正常退出时，保存存储区间列表至磁盘
- 在系统recovery时，恢复存储区间列表。具体恢复流程见异常处理

### 空洞发现

- 空洞发现算法

![绘图](./attachments/1626064228154.drawio.html)


### 空洞填充

![绘图](./attachments/1626068651977.drawio.html)



### 异常处理
###### 主机宕机


###### 网络故障
- 发生网络故障时，比如网络中断/网络超时等，




