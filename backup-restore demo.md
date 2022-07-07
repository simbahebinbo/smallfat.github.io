---
title: backup/restore demo
tags: 
grammar_cjkRuby: true
---
# 步骤
### cluster
 - start cluster，并创建表t1 (启动TA)
```
 	make check PROVE_TESTS='t/tmp_start_cluster.pl'
```
 - 登陆psql
```
	./psql -dpostgres -Usys -p59276 -h127.0.0.1
```
 
 - 插入数据 (psql)
```
	insert into t01 values (100);
```
 
- 基础备份： 
```
 	./pd_backup -dpostgres -w -p57111 -h127.0.0.1 -D /tmp/bak -Xf -Ft -z
```
	
- 插入数据
```
	insert into t01 values (200);
```

- 第一次增量备份
```
	./pd_backup -dpostgres -w -p57111 -h127.0.0.1 -D /tmp/bak -Xf -Ft -z -i
``` 
 
- 插入数据
```
	insert into t01 values (300);
```
 
- 第二次增量备份
```
	./pd_backup -dpostgres -w -p57111 -h127.0.0.1 -D /tmp/bak -Xf -Ft -z -i
``` 
 
- 插入数据
```
	insert into t01 values (400);
```  

- verify backup

- 停止其中一个节点
```
	kill -9 18330
```
  
- 恢复备份数据到已停止节点（ps: 可查看目标数据）
```
 	./pd_restore -D /tmp/bak -t /home/jasonliu/project/postdb2/src/bin/pd_backup/tmp_check/t_205_restore_join_cluster_cluster01_data/ps1 -b 1657098139_base
```

- 启动上述节点
```
    ./pstore -D /home/jasonliu/project/postdb2/src/bin/pd_backup/tmp_check/t_tmp_start_cluster_cluster01_data/ps1 --cluster-name=cluster01 -d 1
```

 ### original
 