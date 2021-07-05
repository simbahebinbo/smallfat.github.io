---
title: aurora paper: understand and think
tags: 新建,模板,小书匠
category: /小书匠/日记/2021-07
renderNumberedHeading: true
grammar_cjkRuby: true
---

# 创新之处
### 内容
- log is database - redo log作为db
- 计算存储分离 - redo log作为存储，从原体系中独立出来
- 计算部分只写IO数据到存储部分的redo log，减少网络延迟

### 原因
- 对于cloud native环境，由于各计算部件、存储部件的虚拟化，网络延迟成为db on clound上影响性能的主要因素。


# 具体设计
### replication

### segment

### shade


### 读写

###### 写
###### 读
###### cache
### 数据一致性
### 故障处理

# 优势
- 存储计算分离：构建跨多中心、自我管理、容错、自我恢复、独立的存储服务，与上层database service故障分离
- database service仅需要写入redo log到storage service，大幅减少网络IO延迟
- backup/restore：TBC