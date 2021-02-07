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
- pstore的存储结构
	- 三级结构： meta1/meta2/user
- range内数据的导入
	- 使用tablet（类似于cockroach的range概念），负责某个范围内的数据的存储查询
	- 生成Sst file
	- 导入Sst file进LSM Tree
- 数据在cluster内的分发
	- 问题：怎么保证cluster内每个node的数据导入均衡问题
	- pstore_service TU: TabletManager
- raft
	- 问题：导入时，巨量数据需要从leader同步到follower，怎么才能保证raft日志不膨胀
	- 
 
### 异常处理
