---
title: postdb - bakcup/restore
tags: 
grammar_cjkRuby: true
---
# 2022.06.24下午 关于backup/restore的会议讨论要点
- 在原生pg_verifybackup中，为什么xlog的verification逐个record进行校验，而不是以文件为单位进行crc校验？
	- 我的推测
		- 基本功能
			- 逐个record验证crc，验证的是每个record的有效性，即该record是否可以正常回放
			- 以文件为单位验证checksum，验证的是这个文件在备份后的完整性
		- 验证效果			
			- 备份文件在备份目录中发生损坏，两种方式都能检测到
			- 若在备份时读取xlog文件内容出现问题（与xlog真实文件内容不一致），此时采用“以文件为单位进行crc校验”这种方法可以通过校验，但record数据实际上已经被破坏了
		- 从验证时间效率来考虑
			- 两者的验证效率差别不大（以crc为例，都需要读取整个文件做checksum）
		- 从版本feature考虑
			- 因manifest清单和pg_verifybackup都是pg13的新feature，也不排除是先做了page data的manifest，而xlog依然复用了老版本的readxlogpage的代码等原因

- backup/restore的安全性
 	- pg_verifybackup利用了manifest中的checksum来做了备份文件的合法性校验，但这个实际上与pg系统（或backup模块）的安全性关系不是很大，这个只是文件的损坏检测，不能防篡改，不能防窃取
	- 系统/backup模块的安全性是另一个维度，如果要实现这个特性，建议通盘考虑

- 增量备份的清单文件
	- 需要在进行增量备份时，记录xlog文件信息

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
- 新建项目pd_restore

----------
# 全局设计
### 备份目录规则
- 按照如下层级组织目录	

``` 
- backup根目录名(自定义)
	|
	|--"$DATETIME1_base"
		 |--具体的备份内容
	|--"$DATETIME2_incremental"
         |--具体的备份内容 
    |--"$DATETIME3_incremental"
         |--具体的备份内容 	
	|--"$DATETIME4_incremental"	
         |--具体的备份内容 	
	|--"$DATETIME5_base"
		 |--具体的备份内容
```

### 备份/恢复关键文件
- 备份信息文件 - 记录base backup/incremental backup的备份信息
  - 每次base backup的\[start_point, end point)
  - 每个base backup包含的所有增量备份信息，并按备份时间升序排列
  	- 每次增量备份目录名称
    - 每次增量备份\[start_point, end point)
  -	最后一次base backup名称  
	
# backup工具的命令行及参数
```
  -D, --pddata=DIRECTORY 备份文件目标根目录
  -F, --format=p|t       base backup时的备份文件输出格式 (plain (default), tar)
  -r, --max-rate=RATE    maximum transfer rate to transfer data directory
                         (in kB/s, or use suffix "k" or "M")
  -T, --tablespace-mapping=OLDDIR=NEWDIR
                         relocate tablespace in OLDDIR to NEWDIR
      --waldir=WALDIR    location for the write-ahead log directory
  -X, --wal-method=none|fetch
                         include required WAL files with specified method
  -z, --gzip             compress tar output
  -Z, --compress=0-9     compress tar output with given compression level


  -c, --checkpoint=fast|spread
                         set fast or spread checkpointing
  -l, --label=LABEL      set base backup label
  -n, --no-clean         do not clean up after errors
  -N, --no-sync          do not wait for changes to be written safely to disk
  -P, --progress         show progress information
  -v, --verbos
  -i  --incremental      指此次备份进行增量备份。若没指定此选项，则进行base backup
  -b  --base=DIRECTORY   若-i 选项已指定，此选项指此次增量备份基于哪次base backup进行
                         若-i 选项已指定，但没指定此选项，则增量备份基于最后一次base backup进行
                         若-i 选项没指定，但指定了此选项，则属非法选项
```



# cluster模式下的单pstore节点数据的basebackup
#### 实现原理与步骤

 1. backup tool 连接到primary node，选择当前NRL最大的pstore节点，获取pstore地址信息
 2. backup tool 连接到pstore node
 3. 备份数据
 4. 备份WAL文件
 5. 验证WAL文件的合法性（为什么要验证：xlogswitch被remove掉以后，存在多线程同时读写wal文件的可能）
	

![绘图](./attachments/1640158663666.drawio.svg)



### 备份数据中的数据一致性
###### data数据
- 在pstore节点上直接做checkpoint, checkpoint结束点作为本次backup的start_point， 该start_point点之前的数据能够确保已经达到当前pgcl且已经落盘
- 在完成checkpoint后，开始备份文件之前，为保证数据文件的完整性应block相应数据落盘，直到备份数据文件完成（因目前FPW功能在postdb中是禁用的，xlog中不带有page image）
	- base backup期间的落盘控制
		- 对buffer进行block的条件：lsn > backup.start_point
		- 对数据buffer进行block需要判断的点位：data page buffer落盘点/clog落盘点/multixact落盘点
		- 如果在backup过程中，replay时产生drop table操作，需要等待backup完成。原因是：drop table操作直接操作数据文件，会造成backup文件的损坏
- 在备份数据文件完成之后，备份WAL文件之前，取backup.end_point = MIN(NCL, PGCL)，由此形成了backup区间 \[start_point, end_point)


###### WAL文件
1. 需要备份\[start_point, end_point)所在的WAL文件
2. 由于在backup过程中，checkpoint完成之后，xlog有可能仍然在持续写入。为了保证seg文件在backup end_point之后无新的record，在WAL文件中backup end_point点之后位置做0填充。

#### 异常处理
###### pstore node不可用
- 在"primary等待ps1回放完成"过程中，若被选中的pstore node(ps1)不可用，则等待会超时，此时通知pg_backup此次backup失败
- 其他时间pstore node不可用，则返回相应的错误给pg_backup，通知此次backup失败

###### primary node不可用
- backup过程中， pg_backup工具与primary node session保持连接的时候，primary node不可用，会导致他们之间的连接断开，backup失败

# cluster模式下单pstore节点数据的restore
  	


![绘图](./attachments/1644887764326.drawio.svg)

# cluster模式下集群的incremental backup
### 需求
- 在base backup的基础上，实现增量备份
- 与原有base backup命令行工具统一，方便使用：base backup工具更名为backup工具，具有base backup/incremental backup两个独立功能
- 支持单独进行增量备份
- 备份目录需要单独指定: 整个backup目录需要统一，增量备份目录是其中的一部分
- 能够让用户方便查看备份历史，包括base backup与incremental backup
- restore数据：新建c项目pd_restore，统一base backup restore与incremental backup restore

### 实现
###### 增量备份序列图如下

![绘图](./attachments/1648606565569.drawio.svg)
###### backup
- 备份起始点start point
	- 定义 - 进行增量备份的起始点
	- 点位值取得 - 从“备份控制文件 backup_info”取得，具体取得方式见“backup工具的命令行及参数”一节

- 目标pstore节点的选择
	- 条件：pstore.NCL >= pstore.PGCL

- 目标WAL文件
	- 同时满足如下条件的WAL文件，表明已经完成写入，文件句柄已被关闭，在增量备份范围内，应当备份
		- seg.end_ptr > start_point
		- seg.end_ptr <= PGCL		
		
	- 满足如下条件的WAL文件，表明正在写xlog，需要备份 \[seg.begin_ptr, PGCL)的部分，其他部分填充0
		- seg.end_ptr > PGCL > seg.begin_ptr

- WAL文件的传送
	- incremental backup模块收集满足如上条件的WAL文件，并发送给backup tool
	- 因pstore节点上WAL文件永远不删除，故总能够取到符合上述条件的WAL文件
	
- WAL文件的合法性验证

###### restore
- 利用pd_restore进行restore；具体请参见本文档restore工具一节

###### backup命令行扩展
- 命令行需要增加的参数
	- 备份类型 - base_backup/incremental_backup
	- 备份根目录 - incremental_backup模式下必须指定。具体目录规则参见“备份目录规则”一节

### 异常情况
###### 网络故障
- backup tool与pstore之间的网络故障
	- backup tool检测到此故障后，清除已备份文件，终止任务，任务失败
	- pstore上检测到此故障后，清除资源，退出当前backend

- backup tool与primary之间的网络故障
	- backup tool检测到此故障后，终止任务，任务失败
	- primary上检测到此故障后，清除资源，退出当前backend

###### 节点故障
- primary 宕机
	- 在backup tool与primary的session存续期间，若primary宕机，backup tool能检测到，此任务会失败
- pstore 宕机
	- 在backup tool与pstore的session存续期间，若pstore宕机，backup tool能检测到，此任务会失败
- backup tool 宕机
 	- pstore/primary检测到后，会清除相应资源，退出backend
	- 启动backup tool后，需要检查backup tool最后一次backup是否为正常退出；否则清理删除最后一次备份的内容与目录
		- 具体实现上，backup目录内设置一个backup运行状态文件backup_status		
		- backup_status的内容：在备份开始前写入此次备份的目录名，并在程序正常退出前清除；
		- 若启动backup tool时发现backup_status文件包含目录名，则表示此次备份过程中backup tool宕机，此目录需要被清除

# restore工具
- 建立一个新的c项目
- 命令行：
```
  pd_restore -b $backup_root_dir -i $backup_name -t $target_dir
```
- 逻辑
  - 从backup_root_dir中找到备份信息文件backup_information
  - 从备份信息文件中查找相应的incremental_backup_name，并找出从base backup到incremental_backup_name的备份路径
  - 验证备份路径中的所有文件的合法性
  - 保存好相应pstore.conf后，清空target_dir
  - 按备份时间升序，解压或拷贝原始数据文件和xlog文件至target_dir指定目录
  - restore完成

# 验证backup文件的完整性
使用verifybackup工具对backup结果文件进行完整性验证

###  验证工具的命令行
```
pd_verifybackup verifies a backup against the backup manifest.

Usage:
  pd_verifybackup [OPTION]... BACKUPDIR

Options:
  -e, --exit-on-error         exit immediately on error
  -i, --ignore=RELATIVE_PATH  ignore indicated path
  -m, --manifest-path=PATH    use specified path for manifest
  -n, --no-parse-wal          do not try to parse WAL files
  -q, --quiet                 do not print any output, except for errors
  -s, --skip-checksums        skip checksum verification
  -w, --wal-directory=PATH    use specified path for WAL files
  -b, --backupname=BACKUP_NAME the source backup name that you want to restore from
  -t, --target=DIRECTORY the target directory that you want to restore to
  -V, --version               output version information, then exit
  -?, --help                  show this help, then exit

```

### 验证流程

![绘图](./attachments/1656050860474.drawio.svg)


# original模式下的backup/restore
postdb还有一个original模式，这个模式下backup/restore的功能要求与原生postgres相同。因此，要考虑与cluster模式下代码的兼容。


----------


# 新设计中的几个疑问
### 去除checkpoint/xlogswitch所导致的方案改动和疑点
- 去除checkpoint record
	- checkpoint.redo之前能确定数据已落盘，base backup以checkpoint.redo作为backup的目标xlog end point
	- 做基于base backup的restore时
		- 原方案 - 从checkpoint.redo点进行replay
		- 问题 - ReadRecord(checkpoint.redo)失败，无法从checkpoint.redo replay，pstore启动失败退出
		- 新方案 - 如果ReadRecord(checkpoint.redo)失败且处于restore阶段，系统正常进入hotstandby模式
- 去除xlogswitch
   - 原方案 - xlogswitch后的xlog文件在被base backup读取时，无其他access操作
   - 问题 - 无xlogswitch时，xlog在被写入的过程中，被base backup读取，内容有无问题？
   - 方案 - backup完成后，检查crc checksum

- 去除backup_lable
   - 原方案 - backup_lable文件用来记录单次base backup的具体信息
   - 问题 - 去除backup_lable后，pstore启动restore时信息从何得来
		- 备份根目录下backup_info记录的是在本目录所有进行backup的信息，不适合作为pstore在restore时的信息
   - 新方案 - 在使用pd_restore工具时，从backup_info内生成一份单次backup信息文件(backup_lable)到目标节点上？

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
