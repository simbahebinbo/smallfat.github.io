---
title: postdb 设计疑问
---

 - MVCC与隔离性是如何共存且统一的？
	 - 不同维度的概念，MVCC侧重描述concurrency control，隔离性定义了一种用来表示数据可见性的语义
 - 事务并发冲突如何解决？
 - HLC在解决分布式事务一致性中起着什么作用？
 - 线性一致性(linearizable)与HLC的关系
	 - linearizable指的是操作的时序与物理时间一致
	 - HLC是逻辑时间与物理时间的混合，不满足linearizable
 - MVCC的实现方法有几类？
	 - 	有代表性的有2类
		 - 	活跃事务表，如pg，就是记录txid大小，以及不同事务对某行数据的insert/update/delete操作，再利用相关rule判断可见性。这个要求在分布式环境下，也许需要对行数据lock才能实现，导致性能问题
		 - 	timestamp - 取得一个全局线性有序的timestamp（依据tx提交时间），然后进行比较，判断出可见性
			 - 	TrueTime From Google
			 - 	HLC from CockroachDB
			 - 	Lamport时钟
 #!/bin/bash
#Minimal implementation of PostgreSQL 9.6+ "non-exclusive" base backup API.

set -e
set -x

TMP_FIFO="/tmp/~tmpfifo_$RANDOM"
TMP_OUT=$(tempfile -d /tmp)
BACKUP_DESTINATION=/tmp

if [ -n "$(jobs)" ] ; then
    echo "Background jobs are running. Please run this script in a different (or sub-) shell. Exiting"
    exit 1
fi

#Make a fifo where psql will listen for the pg_stop_backup() statement:
mkfifo -m 600 $TMP_FIFO

#psql executes a command then waits in the background for data on the fifo:
psql --pset=tuples_only=true --pset=format=unaligned --pset=footer=false -c "select pg_start_backup('my backup', true, false)" -f $TMP_FIFO postgres > $TMP_OUT 2> /dev/null &

#Run rsync or tar here. The exact details will depend on your system. For example:
tar -c --exclude=$PGDATA/postmaster* --exclude=$PGDATA/pg_xlog/* -f $BACKUP_DESTINATION/base_backup.tar $PGDATA

echo "select * from pg_stop_backup(false)" > $TMP_FIFO

#wait for psql to return from background:
wait -n %1

#Treat the whole file as a single line delimited by pipes. Use sed to strip empty lines:
awk 'BEGIN { RS = "\x00" ; FS = "|" } { print $2 }' $TMP_OUT > $BACKUP_DESTINATION/backup_label
awk 'BEGIN { RS = "\x00" ; FS = "|" } { print $3 }' $TMP_OUT | sed -e '/^$/d' > $BACKUP_DESTINATION/tablespace_map

rm -f $TMP_FIFO $TMP_OUT