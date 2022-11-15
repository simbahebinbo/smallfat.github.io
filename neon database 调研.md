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


# 模块

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
nothing to record due to it's crucial part

#### Repository
1. The repository stores all the page versions, or WAL records needed to reconstruct them
2. timeline
	- Each repository consists of multiple Timelines
	- Timeline is a workhorse that accepts page changes from the WAL
	- serves get_page_at_lsn() and get_rel_size() requests
    ***based on above, timeline seems to be workers that record page changes from "WAL redo" process?***
3.  has a WAL redo manager associated with it
	- used to replay PostgreSQL WAL records, whenever we need to reconstruct a page version from WAL to satisfy a GetPage@LSN request, or to avoid accumulating too much WAL for a page
	- it's not related to the WAL redo unit in "WAL Receiver" module

 
#### checkpointing
1. flush WALs from memory to disk in (page-server/safe-keeper)?
2. recycle WAL files in safe-keepers above
3. free up memory that occupied by WALs above

