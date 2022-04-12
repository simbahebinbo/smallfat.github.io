---
title: postdb - bakcup/restore
tags: 
grammar_cjkRuby: true
---
# 2022.03.03下午 关于backup/restore的会议讨论要点
- 针对整个集群的restore：不需要考虑，由dba逐个node实施
- 针对单pstore节点进行restore: 由于pg_restore是pg_dump系的工具，与basebackup不同，无法使用；考虑自己写个简单工具（脚本）
- 针对backup过程中，不好精确控制checkpoint点以后不落盘的问题：不再控制数据落盘；比较checkpoint点与pstore节点上的相应最后落盘点是否相等：若是，则允许备份，否则返回备份失败；
- 针对backup过程中，seg文件存在超过checkpoint点以后的情况，由basebackup工具对checkpoint点以后的数据进行填0处理；相同情况下，sql backup命令则不再被支持；
# 2022.02.15上午 关于backup/restore的会议讨论要点
### 关于backup/restore工具的产品需求与规划
- cluster模式下全量backup工具
	- 针对单pstore节点进行备份: 使用pg_basebackup
	- 针对整个集群备份：不需要
- cluster模式下restore工具
	- 针对单pstore节点进行restore: 使用pg_restore
	- 针对整个集群的restore：考虑使用另外的工具/脚本，调用pg_restore，逐个实现整个cluster的restore
- cluster模式下增量backup工具
	- 参考原生postgres archive机制
	- 暂时不考虑实现
- original模式下backup/restore: 行为要与原生postgres一致

### pg_basebackup工具的设计
- 在primary-node上执行xlogswitch/checkpoint动作，并记录checkpoint 点
- 在primary-node上，只在备份开始时执行一次xlogswitch；备份结束时，不执行xlogswitch
- 在pstore节点上执行backup动作，并发送内容到tool侧
- 在pstore结点上执行backup动作时，不产生任何wal日志，以避免与其他pstore节点上的WAL日志不一致
- 在目标pstore节点上，checkpoint点后的所有数据不落盘，直至backup完成
	- 疑问：若长时间不落盘，page buffer会不会满了丢失数据？（pstore会卡住）
- 备份的数据，截止到checkpoint点 - 由于checkpoint点后的数据不落盘，可以直接拷贝数据目录所有数据
- 备份的wal，截止到checkpoint点 - 即xlogswitch产生的新的segment file之前的所有xlog文件

### pg_restore工具的设计
- 使用bash，对backup文件解压至指定目录

----------
# cluster模式下的单pstore节点数据的basebackup
#### 实现原理与步骤

 1. backup tool 连接到primary node
 2. primary node进行xlog switch，插入XLOGSWITCH日志，生成新的WAL日志文件
 3. primary node进行checkpoint
 4. 所有pstore node回放checkpoint。相应pstore node控制checkpoint点后的数据落盘。primary node等待pstore节点checkpoint回放完成。
 5. primary node向backup tool发送pstore地址和checkpoint location等信息
 6. backup tool根据收到的pstore地址取得pstore node 的sqlport
 7. backup tool 连接到pstore node
 8. 备份数据
 9. 备份WAL文件
	

![绘图](./attachments/1640158663666.drawio.svg)



### 备份数据中的数据一致性
###### data数据
- 比较checkpoint点与pstore节点上的相应最后落盘点是否相等：若是，则允许备份，否则返回备份失败；
- pstore节点上，需要比较的最后落盘点包括：page buffer落盘点/clog落盘点/multixact落盘点

###### seg文件
1. 需要备份checkpoint record所在的seg文件
2. 由于在backup过程中，checkpoint完成之后，xlog有可能仍然在持续写入。为了保证seg文件在checkpoint record之后无新的record，由pg_basebackup在seg文件checkpoint点之后位置做0填充。

#### 异常处理
###### pstore node不可用
- 在"primary等待ps1回放完成"过程中，若被选中的pstore node(ps1)不可用，则等待会超时，此时通知pg_basebackup此次backup失败
- 其他时间pstore node不可用，则返回相应的错误给pg_basebackup，通知此次backup失败

###### primary node不可用
- backup过程中， pg_basebackup工具与primary node session保持连接的时候，primary node不可用，会导致他们之间的连接断开，backup失败
# cluster模式下单pstore节点数据的restore

![绘图](./attachments/1644887764326.drawio.svg)

# cluster模式下集群的incremental backup
### 需求
- 在base backup的基础上，实现增量备份
- 与原有base backup命令行工具统一，方便使用
- 支持在base backup的同时进行增量备份
- 支持单独进行增量备份
- 增量备份时，需要指定目的文件夹
- restore数据时，可利用原生postgresql中的restore_command等选项指定增量备份数据目录

### 实现
###### 增量备份序列图如下
![绘图](./attachments/1648606565569.drawio.svg)

###### 备份模式
- 独立模式 - 在已有base backup的基础上进行incremental backup
- 组合模式 - 先进行base backup，然后在此基础上进行incremental backup

###### 备份逻辑
- checkpoint redo点
	- 定义 - 在进行base backup后，得到的checkpoint的redo点位
	- 点位值取得
		- 组合模式下，从刚完成的base backup取得
		- 独立模式下，由命令行参数指定  

- 目标pstore节点
	- 组合模式下 - 选择base backup同一pstore node
	- 独立模式下 - 任意选择一个当前可用的pstore node

- seg文件的选择
	- 同时满足如下条件的seg文件
		- seg.max_lsn > checkpoint.redo
		- seg.max_lsn <= PGCL

- seg文件的备份
	- incremental backup模块定时收集满足如上条件的seg文件，并发送给backup tool
	- 因xlog文件永远不删除，故总能够取到符合上述条件的seg文件

###### restore
- 利用原生postgresql中的restore_command等选项指定增量备份数据目录
- 进行restore 回放时，增量备份数据目录中的xlog文件优先于pg_wal目录中的xlog文件，因此base backup中pg_wal下的被截断(尾部填充为0)的xlog不影响restore replay

###### 终止增量备份
- 用户在backup tool端，按下ctrl-c时，终止增量备份
- 此时，backup tool发送end_incremental_backup请求给pstore node；pstore node在完成当前seg文件传送后，退出incremental backup模块，关闭background

###### backup命令行扩展
- 命令行需要增加的参数
	- 备份模式 - base_backup/incremental_backup/组合模式
	- 增量备份目标目录 - incremental_backup模式下必须指定
	- checkpoint_redo点 - incremental_backup模式下必须指定

### 异常情况
###### 网络故障
- backup tool与pstore之间的网络故障
	- backup tool检测到此故障后，清除已备份文件，终止任务，任务失败
	- pstore上检测到此故障后，清除资源，退出当前background

- backup tool与primary之间的网络故障
	- backup tool检测到此故障后，终止任务，任务失败
	- primary上检测到此故障后，清除资源，退出当前background

###### 节点故障
- primary 宕机
	- 在backup tool与primary的session存续期间，若primary宕机，backup tool能检测到，此任务会失败
- pstore 宕机
	- 在backup tool与pstore的session存续期间，若pstore宕机，backup tool能检测到，此任务会失败
- backup tool 宕机
 	- pstore/primary检测到后，会清除相应资源，退出background

# cluster模式下集群的restore（经讨论，无需考虑，由用户决定）
- 由于现在的备份策略是只备份checkpoint点之前的数据，整个集群所有node的数据都会维持在这个点，因此cluster的restore就相当于复制n个备份数据集，并替换各自的配置文件，然后启动primary node
- 建议使用script完成这个功能
- pg_restore是属于pg_dump类的工具，不能用于basebackup类的备份数据

# original模式下的backup/restore
postdb还有一个original模式，这个模式下backup/restore的功能要求与原生postgres相同。因此，要考虑与cluster模式下代码的兼容。


----------


# 新设计中的几个疑问
### 原始stop backup动作
- request_xlog_switch - 保证archiver立即能够拷贝当前seg文件，使得backup快速结束
- insert XLOG_BACKUP_END record - stop_point = insert处lsn，在回放XLOG_BACKUP_END record时设置miniRecoveryPoint/backupStartPoint

### 原始archive与basebackup的关系
- basebackup时，要等到执行backup过程中的xlog都被archive了，才结束backup
- 这意味着除了checkpoint所在的segment中的record，之后的record都将被忽略

# restore时的关键变量
- standbymode
```
`````````
- minRecoveryPoint
```
	 * minRecoveryPoint is updated to the latest replayed LSN whenever we
	 * flush a data change during archive recovery. That guards against
	 * starting archive recovery, aborting it, and restarting with an earlier
	 * stop location. If we've already flushed data changes from WAL record X
	 * to disk, we mustn't start up until we reach X again. Zero when not
	 * doing archive recovery.
```
- backupStartPoint
```
  	 * backupStartPoint is the redo pointer of the backup start checkpoint, if
	 * we are recovering from an online backup and haven't reached the end of
	 * backup yet. It is reset to zero when the end of backup is reached, and
	 * we mustn't start up before that. A boolean would suffice otherwise, but
	 * we use the redo pointer as a cross-check when we see an end-of-backup
	 * record, to make sure the end-of-backup record corresponds the base
	 * backup we're recovering from.
```
 
- ArchiveRecoveryRequested
```
/*
 * When ArchiveRecoveryRequested is set, archive recovery was requested,
 * ie. signal files were present. When InArchiveRecovery is set, we are
 * currently recovering using offline XLOG archives. These variables are only
 * valid in the startup process.
 *
```
 
- InArchiveRecovery
 
 ```
 * When ArchiveRecoveryRequested is true, but InArchiveRecovery is false, we're
 * currently performing crash recovery using only XLOG files in pg_wal, but
 * will switch to using offline XLOG archives as soon as we reach the end of
 * WAL in pg_wal.
```

============
# pstore实现：WAL Record/XLOG文件写入
### fsync/fdatasync/fflush
### XLogInsertRecord
# pstore实现：StartupXLOG
### Standby模式下，StartupXLOG依靠一个循环体完成系统回放的动作
- 根据不同的场景，分别从PGWAL/ARCHIVE/STREAM三个data source获取WAL
- 有三个场景
	- SHUTDOWN
	- CRASH_RECOVERY
	- ARCHIVE_RECOVERY

```
XLogFileRead xlog.c:3664
WaitForWALToBecomeForStorageNode xlog.c:14066
WaitForWALToBecomeAvailable xlog.c:13274
XLogPageRead xlog.c:13114
ReadPageInternal xlogreader.c:609
XLogReadRecord xlogreader.c:330
ReadRecord xlog.c:4309
ReadCheckpointRecord xlog.c:9356
StartupXLOG xlog.c:7744
StartupProcessMain startup.c:200
AuxiliaryProcessMain bootstrap.c:517
co_start_auxiliary_process ps_main.c:4135
__lambda20::operator() libgo_c.cc:36
std::_Function_handler::_M_invoke(const std::_Any_data &) functional:2071
std::function::operator()() const functional:2471
co::Task::__lambda2::operator() task.cpp:82
co::Task::Run task.cpp:93
co::Task::StaticRun task.cpp:124
make_fcontext make_x86_64_sysv_elf_gas.S:64
<unknown> 0x0000000000000000
```
