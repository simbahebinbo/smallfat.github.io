---
title: postdb v4 - postdb cluster service 高层设计
grammar_cjkRuby: true
---
# 背景与目标


# 在postdb-v4系统中的位置

# 状态图

## pcs node 状态

![绘图](./attachments/1670310960410.drawio.svg)

- pcs_recovery
- pcs_election
- pcs_service_available

# 功能

## pcs 选举

![enter description here](./images/Screenshot_from_2022-12-07_09-43-40.png)

## cluster node状态管理

- primary pcs利用心跳机制定时收集cluster内各node的状态，包括：
	- 在线情况
	- 其他业务指标(shard数，shard总占用资源，io负载，cpu负载，memory占用，硬盘空闲容量等)
- 此状态信息保存在内存cache中，无需持久化，无需被复制到replica pcs: 切换primary pcs场景下，新的primary pcs会重新收集最新cluster node最新状态信息

## 系统元信息
### 元信息类别
- shard 元信息: shard id/key range/primary node/replica nodes
- shard-group 元信息：shard group id/
- 配置信息

### 元信息同步机制

#### 使用raft协议进行元信息同步

![绘图](./attachments/1670395352769.drawio.svg)
#### 效率


## shard/shard group管理

### 注册shard信息
### 注销shard信息

## 扩容/缩容

## 控制命令下发

# 接口

# 容错机制

