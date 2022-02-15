---
title: postdb - bakcup/restore
tags: 
grammar_cjkRuby: true
---
# cluster模式下的单pstore节点数据的backup
- 原生postgres内有两种进行数据backup的方法
	- "SELECT pg_start_backup" / "SELECT * FROM pg_stop_backup"
	- pg_basebackup工具
- 以下以pg_basebackup工具为例，说明cluster模式下backup实现原理：
	- 目标pstore node的选择方法有两种
		- 在命令行参数栏指定 - 当前采用方法，但无法兼容原生pg 命令行
		- 由系统自动选择：收集系统archive mode参数/(ip,sqlport)参数，若有pstore node设置该参数为on/always，则该pstore node为目标pstore node；若没有任何pstore node的archive mode设为on/always，则任选一个pstore node
	- primary做checkpoint后，需要等待被选择的pstore node完成此动作（即达到NRL）
	- backup tool要使用pstore的ip/sqlport来完成连接到pstore node的动作


![绘图](./attachments/1640158663666.drawio.svg)


# cluster模式下单pstore节点WAL日志的archive
原生postgres已经含有WAL日志的archive功能。cluster模式下，对于单pstore节点的WAL日志的archive，通过设置"archive"相关参数后重启pstore，也能达到相同目的。

# cluster模式下单pstore节点数据的restore


![绘图](./attachments/1644887764326.drawio.svg)

# original模式下的backup/restore
postdb还有一个original模式，这个模式下backup/restore的功能要求与原生postgres相同。因此，要考虑与cluster模式下代码的兼容。
