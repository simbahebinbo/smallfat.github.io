---
title: IMPORT功能调研
renderNumberedHeading: true
grammar_cjkRuby: true
---
# 目的与现状

# 调研结果
### SQL Layer 
###### 从CSV文件生成KV格式数据

###### 导入语法的格式以及Parser

### Storage Layer 
###### 从KV数据生成SST文件
- SstFileWriter

###### 将SST文件导入RocksDB LSM Tree
- IngestExternalFile

###### 分布式存储
- tablet
- pslib
- raft
 

