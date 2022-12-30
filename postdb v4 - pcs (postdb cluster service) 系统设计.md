---
title: postdb v4 - pcs (postdb cluster service) 系统设计
tags: 
grammar_cjkRuby: true
---
# 背景与目标
- pcs是postdb数据库下的一个模块
- pcs需要收集postdb cluster中所有node的在线状态(通过网络层)
- pcs需要在创建table/index/partition等relation时，创建对应的分片(shard)，并根据策略进行配置(副本策略，primary shard node策略)
- pcs需要在drop table/index/partition等relation时，回收对应的分片(shard)
- pcs需要在alter table/index/partition等relation时，处理对应的分片(shard)
- pcs需要应对失效的cluster node(计算节点/存储节点)，进行failover处理
- pcs需要应对shard 容量膨胀情况，进行自动分裂
- pcs需要应对shard 写过热问题，进行自动分裂/平移
- pcs需要支持shard group配置，用户指定的table 中所有shard 存储位置在一起
- pcs需要支持控制命令(日志命令/tracing命令/监控命令等)下发给指定node

# 在系统总体架构中的位置

![enter description here](./images/postdbv4-arch.png)


# 状态迁移

![绘图](./attachments/1672372460895.drawio.svg)


# 功能模块

## PCS group

![绘图](./attachments/1672372540325.drawio.svg)
- 整个cluster中，pcs 用来控制整个cluster的状态变化。使用pcs group来保证pcs node的可用性
- pcs group之间采用与Raft类似的协议，选举primary pcs节点，并在pcs之间同步元信息


### pcs 选举
- pcs 节点在进行选举时，状态迁移如下图所示：

![enter description here](./images/Screenshot_from_2022-12-07_09-43-40.png)
- 节点选举是在pcs 节点之间进行的，pcs 节点是candidate 与 follower


### pcs间元信息同步


## 元信息管理


## shard调度管理

## 配置下发

## 控制命令下发

## cluster node状态管理
