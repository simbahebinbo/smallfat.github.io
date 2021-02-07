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
- range内数据的处理
	- 使用tablet（类似于cockroach的range概念），负责某个范围内的数据的存储查询
	- 生成Sst file
	- 导入Sst file进LSM Tree
- 数据在cluster内的分发
	- 
- raft
 
### 异常处理
