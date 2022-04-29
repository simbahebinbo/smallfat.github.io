---
title: Data Page 1 - tuple内幕
tags: 
grammar_cjkRuby: true
---
# Storage
![图：数据文件格局](./images/Screenshot_from_2022-04-20_20-51-06.png)


### Relations
- table
- sequence
- materialized view
- index


### Folks
- main
- fsm
- vm

### Files
- 在磁盘上的数据文件路径
	- 目录：$tablespace_name/$database_oid
	- 文件
		- $relation_oid/$relation_oid.1......
		- $relation_oid.fsm/$relation_oid.fsm.1......
		- $relation_oid.vm

### Pages
- 每个数据文件，内部由pages组成。如图“图：数据文件格局”
# Page
### Page Layout

![page inner layout](./images/Screenshot_from_2022-04-20_16-39-35.png)

![table - page - tuple三级关系图](./images/1650894154257.png)
### Page Header
![page header field图](./images/1650895897771.png)

- pd_lsn - 当前page内最新的数据所对应的xlog lsn
- pd_checksum
- pd_flags
- pd_lower - 指向空闲区起始地址
- pd_upper - 指向空闲区结束地址
- pd_special
- pd_pagesize_version
- pd_prune_xid


### Line pointer
- 

### Tuple Header
![fields in tuple header](./images/1650894403055.png)

### Tuple Data

# MVCC
