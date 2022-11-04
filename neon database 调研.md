---
title: neon database 调研
tags: 
grammar_cjkRuby: true
---
# 基本信息
- source code: https://github.com/neondatabase/neon

# 架构
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

### Services

![enter description here](./images/Screenshot_from_2022-11-04_13-12-27.png)

S3 is the main fault-tolerant storage of all data, as there are no Page Server replicas. We use a separate fault-tolerant WAL service to reduce latency. It keeps track of WAL records which are not synced to S3 yet.

The Page Service listens for GetPage@LSN requests from the Compute Nodes, and responds with pages from the repository. On each GetPage@LSN request, it calls into the Repository function

A separate thread is spawned for each incoming connection to the page service. The page service uses the libpq protocol to communicate with the client. The client is a Compute Postgres instance

# WAL service

![enter description here](./images/Screenshot_from_2022-11-04_13-09-03.png)

