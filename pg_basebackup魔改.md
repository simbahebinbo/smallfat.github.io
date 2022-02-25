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
	- 疑问：若长时间不落盘，page buffer会不会满了丢失数据？
- 备份的数据，截止到checkpoint点 - 由于checkpoint点后的数据不落盘，可以直接拷贝数据目录所有数据
- 备份的wal，截止到checkpoint点 - 即xlogswitch产生的新的segment file之前的所有xlog文件

### pg_restore工具的设计

# cluster模式下的单pstore节点数据的backup
- 原生postgres内有两种进行数据backup的方法
	- "SELECT * from pg_start_backup" / "SELECT * FROM pg_stop_backup"
	- pg_basebackup工具
- 以下以pg_basebackup工具为例，说明cluster模式下backup实现原理：
	- 目标pstore node的选择方法有两种
		- 在命令行参数栏指定 - 当前采用方法，但无法兼容原生pg 命令行
		- 由系统自动选择：收集系统archive mode参数/(ip,sqlport)参数，若有pstore node设置该参数为on/always，则该pstore node为目标pstore node；若没有任何pstore node的archive mode设为on/always，则任选一个pstore node
	- primary做checkpoint后，需要等待被选择的pstore node完成此动作（即达到NRL）
	- backup tool要使用pstore的ip/sqlport来完成连接到pstore node的动作



![绘图](./attachments/1640158663666.drawio.svg)
### 问题
由于archive/backup/restore在原生的设计中，联系紧密。现在的设计把base backup独立考虑，可能会对archive/restore后续造成影响

# cluster模式下单pstore节点WAL日志的archive
原生postgres已经含有WAL日志的archive功能。cluster模式下，对于单pstore节点的WAL日志的archive，通过设置"archive"相关参数后重启pstore，也能达到相同目的。

# cluster模式下单pstore节点数据的restore


![绘图](./attachments/1644887764326.drawio.svg)
### 问题
standbymode下的restore，如果pg_wal或者archive没有后续record，或者有invalid record，会切换到stream source等待，造成StartupXLOG无法结束。
# original模式下的backup/restore
postdb还有一个original模式，这个模式下backup/restore的功能要求与原生postgres相同。因此，要考虑与cluster模式下代码的兼容。

=========================

# 新设计中的几个疑问
### 原始stop backup动作
- request_xlog_switch - 保证archiver立即能够拷贝当前seg文件，使得backup快速结束
- insert XLOG_BACKUP_END record - stop_point = insert处lsn，在回放XLOG_BACKUP_END record时设置miniRecoveryPoint/backupStartPoint

### 去掉stop backup中的request_xlog_switch动作引起的问题
- checkpoint 所在的xlog就不会被backup，也不会被archive

### 去掉stop backup中的XLOG_BACKUP_END引起的问题
- minRecoveryPoint在回放结束时没有被设置

### restore时的关键变量
- standbymode
```

```
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
 ``````
 
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

