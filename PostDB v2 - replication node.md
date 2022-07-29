---
title: postdb - replication node
tags: postdb2 replication node
category: /小书匠/日记/2022-07
grammar_cjkRuby: true
---
# 原始设计文档
 ![replic设计说明_v3](./attachments/replic设计说明_v3.docx)
 
 # 重点
 ### Primary/Replication之间的数据一致性的涵义
 - wal一致性 - 有限时间内的相对一致性
 - 传送的wal必须是有效的 - 达到PGCL/NCL条件的wal才允许被传送
 
 ### Primary/Replication状态机迁移
 
 ### 
 
 # 问题疑惑
 