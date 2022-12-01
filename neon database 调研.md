---
title: neon database 调研
tags: 
grammar_cjkRuby: true
---
# 基本信息
- source code: https://github.com/neondatabase/neon

# 架构

![enter description here](./images/Screenshot_from_2022-11-04_13-09-03.png)


## major components:
#### Pageserver

Scalable storage backend for the compute nodes.

 - Repository - Neon storage implementation.
 - WAL receiver - service that receives WAL from WAL service and stores it in the repository.
 - Page service - service that communicates with compute nodes and responds with pages from the repository.
 - WAL redo - service that builds pages from base images and WAL records on Page service request

#### WAL service

The service receives WAL from the compute node and ensures that it is stored durably.


# components

## Page Server

The Page Server has a few different duties:

 - Respond to GetPage@LSN requests from the Compute Nodes
 - Receive WAL from WAL safekeeper, and store it
 - Upload data to S3 to make it durable, download files from S3 as needed

S3 is the main fault-tolerant storage of all data, as there are no Page Server replicas. We use a separate fault-tolerant WAL service to reduce latency. It keeps track of WAL records which are not synced to S3 yet.


### Services

The Page Server consists of multiple threads that operate on a shared repository of page versions:

![enter description here](./images/Screenshot_from_2022-11-04_13-12-27.png)
#### Page Service

 1. listens for GetPage@LSN requests from the Compute Nodes
 2. responds with pages from the repository
 3. On each GetPage@LSN request, it calls into the Repository function


#### WAL Receiver

 1. connects to the external WAL safekeeping service
 2. continuously receives WAL
 3. decodes the WAL records, and stores them to the repository. ***(but it's "WAL REDO''s duty?)***

#### Backup Service
nothing to record due to it's not crucial part

#### Repository

##### logic
1. The repository stores all the page versions, or WAL records needed to reconstruct them
2. timeline
	- Each repository consists of multiple Timelines
	- Timeline is a workhorse that accepts page changes from the WAL
	- serves get_page_at_lsn() and get_rel_size() requests
    ***based on above, timeline seems to be workers that record page changes from "WAL redo" process?***
3.  has a WAL redo manager associated with it
	- used to replay PostgreSQL WAL records, whenever we need to reconstruct a page version from WAL to satisfy a GetPage@LSN request, or to avoid accumulating too much WAL for a page
	- it's not related to the WAL redo unit in "WAL Receiver" module

##### storage subsystem
```
Cloud Storage                   Page Server                           Safekeeper
                        L1               L0             Memory            WAL

+----+               +----+----+
|AAAA|               |AAAA|AAAA|      +---+-----+         |
+----+               +----+----+      |   |     |         |AA
|BBBB|               |BBBB|BBBB|      |BB | AA  |         |BB
+----+----+          +----+----+      |C  | BB  |         |CC
|CCCC|CCCC|  <----   |CCCC|CCCC| <--- |D  | CC  |  <---   |DDD     <----   ADEBAABED
+----+----+          +----+----+      |   | DDD |         |E
|DDDD|DDDD|          |DDDD|DDDD|      |E  |     |         |
+----+----+          +----+----+      |   |     |
|EEEE|               |EEEE|EEEE|      +---+-----+
+----+               +----+----+
```

###### the path to store WAL
1. memory phase
	-  WAL is received as a stream from the Safekeeper
	-  captured by the page server and stored quickly in memory
	-  keep the WAL records for the same page and relation close to each other, can be thought of as a quick "reorder buffer" 
		- 要解析每个wal record，得到page/relation
		- 根据page/relation，将wal record插进合适的位置

2. L0 phase
	- enough WAL has been accumulated, it is flushed to disk into a new L0 layer file
	- covers the whole key range is called a L0 file 
```
    000000067F000032BE0000400000000020B6-000000067F000032BE0000400000000030B6__000000578C6B29-0000000057A50051
              start key                          end key                          start LSN     end LSN
```


3. L1 phase
	- When enough L0 files have been accumulated, they are ***merged*** together and ***sliced*** per key-space
	- 此处的key是指merge以后，形成的segment file内wal record的地址
	- covers only part of the key range is called a L1 file
	
 ```
     000000000000000000000000000000000000-FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF__000000578C6B29-0000000057A50051
 ```
 
 
###### Layer

1. state
- In-memory
- on-disk
- others
	- open
	- closed
	- frozen

2. kinds
- image file 
    contains a ***snapshot*** of all keys at a particular LSN
	

```
    000000067F000032BE0000400000000070B6-000000067F000032BE0000400000000080B6__00000000346BC568
              start key                          end key                           LSN
```
	
	
- delta file
	contains modifications to a segment - mostly in the form of WAL records - in a range of LSN

```
000000067F000032BE0000400000000020B6-000000067F000032BE0000400000000030B6__000000578C6B29-0000000057A50051
```
 
#### checkpointing

1. flush WALs from memory to disk in page-server?
2. recycle WAL files in safe-keepers above
3. free up memory that occupied by WALs above

# image file details
## related data structures

```rust
///
/// Represents a partitioning of the key space.
///
/// The only kind of partitioning we do is to partition the key space into
/// partitions that are roughly equal in physical size (see KeySpace::partition).
/// But this data structure could represent any partitioning.
///
pub struct KeyPartitioning {
    pub parts: Vec<KeySpace>,
}

pub struct Timeline {
    pub tenant_id: TenantId,
    pub timeline_id: TimelineId,
    pub layers: RwLock<LayerMap>,
    last_freeze_at: AtomicLsn,
    // Atomic would be more appropriate here.
    last_freeze_ts: RwLock<Instant>,

    // WAL redo manager
    walredo_mgr: Arc<dyn WalRedoManager + Sync + Send>,

    // What page versions do we hold in the repository? If we get a
    // request > last_record_lsn, we need to wait until we receive all
    // the WAL up to the request. The SeqWait provides functions for
    // that. TODO: If we get a request for an old LSN, such that the
    // versions have already been garbage collected away, we should
    // throw an error, but we don't track that currently.
    //
    // last_record_lsn.load().last points to the end of last processed WAL record.
    //
    // We also remember the starting point of the previous record in
    // 'last_record_lsn.load().prev'. It's used to set the xl_prev pointer of the
    // first WAL record when the node is started up. But here, we just
    // keep track of it.
    last_record_lsn: SeqWait<RecordLsn, Lsn>,

    // All WAL records have been processed and stored durably on files on
    // local disk, up to this LSN. On crash and restart, we need to re-process
    // the WAL starting from this point.
    //
    // Some later WAL records might have been processed and also flushed to disk
    // already, so don't be surprised to see some, but there's no guarantee on
    // them yet.
    disk_consistent_lsn: AtomicLsn,

    // Parent timeline that this timeline was branched from, and the LSN
    // of the branch point.
    ancestor_timeline: Option<Arc<Timeline>>,
    ancestor_lsn: Lsn,

    // Metrics
    metrics: TimelineMetrics,

    /// If `true`, will backup its files that appear after each checkpointing to the remote storage.
    upload_layers: AtomicBool,

    /// Ensures layers aren't frozen by checkpointer between
    /// [`Timeline::get_layer_for_write`] and layer reads.
    /// Locked automatically by [`TimelineWriter`] and checkpointer.
    /// Must always be acquired before the layer map/individual layer lock
    /// to avoid deadlock.
    write_lock: Mutex<()>,

    /// Used to avoid multiple `flush_loop` tasks running
    flush_loop_started: Mutex<bool>,

    /// layer_flush_start_tx can be used to wake up the layer-flushing task.
    /// The value is a counter, incremented every time a new flush cycle is requested.
    /// The flush cycle counter is sent back on the layer_flush_done channel when
    /// the flush finishes. You can use that to wait for the flush to finish.
    layer_flush_start_tx: tokio::sync::watch::Sender<u64>,
    /// to be notified when layer flushing has finished, subscribe to the layer_flush_done channel
    layer_flush_done_tx: tokio::sync::watch::Sender<(u64, anyhow::Result<()>)>,

    /// Layer removal lock.
    /// A lock to ensure that no layer of the timeline is removed concurrently by other tasks.
    /// This lock is acquired in [`Timeline::gc`], [`Timeline::compact`],
    /// and [`Tenant::delete_timeline`].
    layer_removal_cs: Mutex<()>,

    // Needed to ensure that we can't create a branch at a point that was already garbage collected
    pub latest_gc_cutoff_lsn: Rcu<Lsn>,

    // List of child timelines and their branch points. This is needed to avoid
    // garbage collecting data that is still needed by the child timelines.
    pub gc_info: RwLock<GcInfo>,

    // It may change across major versions so for simplicity
    // keep it after running initdb for a timeline.
    // It is needed in checks when we want to error on some operations
    // when they are requested for pre-initdb lsn.
    // It can be unified with latest_gc_cutoff_lsn under some "first_valid_lsn",
    // though let's keep them both for better error visibility.
    pub initdb_lsn: Lsn,

    /// When did we last calculate the partitioning?
    partitioning: Mutex<(KeyPartitioning, Lsn)>,

    /// Configuration: how often should the partitioning be recalculated.
    repartition_threshold: u64,

    /// Current logical size of the "datadir", at the last LSN.
    current_logical_size: LogicalSize,
    initial_size_computation_started: AtomicBool,

    /// Information about the last processed message by the WAL receiver,
    /// or None if WAL receiver has not received anything for this timeline
    /// yet.
    pub last_received_wal: Mutex<Option<WalReceiverInfo>>,

    /// Relation size cache
    pub rel_size_cache: RwLock<HashMap<RelTag, (Lsn, BlockNumber)>>,

    state: watch::Sender<TimelineState>,
}
```

## file structure

## everything of key

## everything of value

## when and how to generate image		
	

# Q & A

1. For the database system, when is the write operation done?
	- a. WAL写入safekeeper成功，并达到多数？ b. pageserver 回放了WAL，且持久化成功?
	- 采用方案a，应该能更快提高写的效率。读数据的时候，若相应wal还没有回放，则读请求等待。safekeeper与计算端之间有协议
	
2. pageserver: memory - L0 - L1多级存储结构，设计意图是什么？
	- 类似于LSM-Tree思想
		- 在memory层存储已经排序(按relation/page)的wal record
		- memory层积累到一定程度，持久化到disk，形成L0文件
		- L0层慢慢的有了多个L0文件，积累到一定程度，合并成L1文件，并持久化到L1层
	- 优缺点
		- safekeeper向pageserver写WAL时，wal record放进memory层后，该写操作就可以认为完成了，提高了写效率
		- L0层的持久化是对数据块的append，写效率高
		- 从存储读数据，要找到对应区间的image file和delta file
	

3. image file和delta file怎么做到快速取得指定lsn处的数据?
	- 基于image file，回放delta file至指定lsn

4. branch的设计，有什么益处呢？
	- 对这种类型的数据：数据从某个lsn开始不同，可以新开一个timeline，即branch，大大提高了写效率，节约了磁盘空间

5. pageserver在能否实现并行回放wal以提高回放效率？
	- 数据库多个不同的表，或者同一个表下多个不同的page同时进行写操作，可以并行回放，提高效率

6. 在存储部分实现shard存储，有什么益处和缺点？

7. 如何在存储部分实现shard存储？

8. L1文件的lsn range和key range是什么关系？



