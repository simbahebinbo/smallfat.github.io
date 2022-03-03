---
title: postdb - bakcup/restore
tags: 
grammar_cjkRuby: true
---
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
==================================================================================================
# cluster模式下的单pstore节点数据的backup
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

#### 控制数据落盘
pstore node上，在checkpoint完成后，进行备份之前，我们需要保证checkpoint location点后的数据不再落盘。这是由于FullPageWrite功能被全局关闭了，否则可能造成错误数据。在备份完成后，再打开落盘开关。

###### 步骤
1. primary node先选择一个active的pstore节点ps1
2. primary node生成新的checkpoint日志，并在record中记录ps1的node id，其位置lsn - 这里的日志标志应该是一个新的CHECKPOINT类型XLOG_CHECKPOINT_BACKUP，否则pstore无法与其他checkpoint区分
3. 所有pstore node回放到此日志时， 根据node id判断是否本pstore node，如果是的话，该存储结点的bufmgr里面只要大于这个checkpoint lsn就不落盘，卡住
4. primary等待ps1回放完成
5. primary node通知pg_basebackup哪个存储结点被选中了(这里是ps1)
6. pg_basebackup在ps1存储结点进行备份
7. ps1完成备份后，打开落盘开关，继续落盘

#### XLOG文件的备份


#### 异常处理
###### 选定的pstore node不可用
###### primary node不可用
# cluster模式下单pstore节点数据的restore

![绘图](./attachments/1644887764326.drawio.svg)

# original模式下的backup/restore
postdb还有一个original模式，这个模式下backup/restore的功能要求与原生postgres相同。因此，要考虑与cluster模式下代码的兼容。

=========================

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
