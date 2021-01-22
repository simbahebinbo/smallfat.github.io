---
title: RocksDB
tags: 
renderNumberedHeading: true
grammar_cjkRuby: true
---
# 前言
以下是关于rocksdb的一些事实：
- fork自level db
- 在level db的基础上增加了一些feature
- 由facebook开发
- 开发语言是C++
- 继承了level db的memtable , sst等机制

# 架构模块
### 架构层次
##### 写数据
![写数据-图片来自网络](https://gitee.com/string_coder/xiaoshujiang/raw/master/write.jpg)

如上图，写入时
 - 数据会被追加到memtable
 	 - memtable位于内存区域。
	 - 由此可见，对于写线程来说，写入内存即可返回，大大提高了写的速度

##### 读数据
![读数据-图片来自网络](https://gitee.com/string_coder/xiaoshujiang/raw/master/read.jpg)

### 功能点概览
![功能点-图片来自网络](https://gitee.com/string_coder/xiaoshujiang/raw/master/functions.png)


# 重点特性
