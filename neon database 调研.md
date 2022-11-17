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
	-  covers the whole key range is called a L0 file 
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
	
	1. wal record or page data？
	2. how to generate?
	3. file name format: 
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

# Questions
1. For the database system, when is the write operation done?
	- WAL写入safekeeper成功，并达到多数？
	- pageserver 回放WAL成功
	- 