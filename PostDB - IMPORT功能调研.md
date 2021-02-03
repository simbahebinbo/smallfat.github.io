---
title: IMPORT功能调研
renderNumberedHeading: true
grammar_cjkRuby: true
---
# 目的与现状

# 调研结果
### SQL Layer 
###### 从CSV文件生成KV格式数据
- 可自己实现

###### Copy导入语法的AST分析

### Storage Layer 
###### 从KV数据生成SST文件
- SstFileWriter

###### 将SST文件导入RocksDB LSM Tree
- IngestExternalFile



