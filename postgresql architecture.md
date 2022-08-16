---
title: postgresql architecture
tags: 
category: 
renderNumberedHeading: true
grammar_cjkRuby: true
---

# Architecture
![postgres layed architecture](https://www.postgresql.fastware.com/hs-fs/hubfs/Images/Diagrams/img-dgm-postgresql-shared-memory.png?width=1000&name=img-dgm-postgresql-shared-memory.png)

![postgres components relationship](https://oscimg.oschina.net/oscnet/54f21979bf63cdb48f38312b368c2bca2cb.jpg)


# Process
![enter description here](https://raw.githubusercontent.com/smallfat/smallfat.github.io/master/小书匠/1660661807723.png)


# Memory
![enter description here](https://raw.githubusercontent.com/smallfat/smallfat.github.io/master/小书匠/1660661713149.png)

- 数据共享缓冲区：PostgreSQL把要操作和处理的表、index，读入到内存中，放到该区域缓存。
- 日志缓冲区：用于缓存数据库中对数据修改的日志记录，如：update table test set id=1这条SQL语句，数据库会把这个操作的信息记录在该内存区，将来写出到日志文件中，如果配置为归档模式，则最终写出到归档日志文件中去，用于恢复使用。其大小由wal_buffers参数决定。
- 提交日志缓冲区：该内存区域有别于wal buffer日志缓冲区。它用于记录数据库中所有事务的提交状态，事务是否已经提交，是否已经终止，是否进行中，子事务等状态信息。用于MVCC。


# Storage
### Logical Storage
![enter description here](https://raw.githubusercontent.com/smallfat/smallfat.github.io/master/小书匠/1660661223795.png)

### Physical Storage
![enter description here](https://raw.githubusercontent.com/smallfat/smallfat.github.io/master/小书匠/1660661265169.png)

# Lock Mechanism
