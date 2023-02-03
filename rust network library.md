---
title: rust network library
tags: 
grammar_cjkRuby: true
---
# 2023.1.20

![绘图](./attachments/1674173629455.drawio.svg)


- 待办
1 大数据量测试及优化
2. message定义在lib库内：应在lib外应用层定义
3. 读写分离，分别在各自的task内运行

# 2023.2.3

![绘图](./attachments/1675396183130.drawio.svg)


1. message定义在lib库内：应在lib外应用层定义
2. 读写分离，分别在各自的task内运行
3. 测试：
pingpong测试
大数据量测试
raft选举
