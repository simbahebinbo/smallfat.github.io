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
 