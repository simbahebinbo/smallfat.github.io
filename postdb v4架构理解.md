---
title: postdb v4 架构理解
tags: 
grammar_cjkRuby: true
---
# 需求
## V3版本中存在的问题

## V4的目标


# 本地化架构、理解、问题

![enter description here](./images/Screenshot_from_2022-11-28_15-34-37.png)

## 初始化流程

## 写流程

## 读流程

## question
1. sharding的key：wal lsn？
2. node变化(增加/减少)引起sharding key range的变化，要及时推送到存储层，重构存储
3. sharding元信息保存在pcs内存，并持久化到disk

