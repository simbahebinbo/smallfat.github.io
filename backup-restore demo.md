---
title: backup/restore demo
tags: 
grammar_cjkRuby: true
---
# 基本步骤

 1. start cluster  (by TA)
 2. create table  (psql)
 3. insert 1
 4. pd_backup -d postgres -w -p 57111 -D /tmp/bak -Xf -Ft -z
 5. insert 2
 6. pd_backup -d postgres -w -p 57111 -D /tmp/bak -Xf -Ft -z -i
 7. insert 3
 8. pd_backup -d postgres -w -p 57111 -D /tmp/bak -Xf -Ft -z -i
 9. insert 4
 10. pd_backup -d postgres -w -p 57111 -D /tmp/bak -Xf -Ft -z -i
 11. pd_restore -D /tmp/bak -t /home/jasonliu/project/postdb2/src/bin/pd_backup/tmp_check/t_205_restore_join_cluster_cluster01_data/ps1 -b 1657098139_base
 11. check