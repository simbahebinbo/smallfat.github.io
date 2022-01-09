---
title: PostDB - wal日志空洞管理模块设计
tags: WAL 
category: /小书匠/日记/2021-07
renderNumberedHeading: true
grammar_cjkRuby: true
---


# 功能
### 总体功能

![绘图](./attachments/1626283921116.drawio.html)

###### 模式
本模块在不同的场景下，分别工作于以下模式下：普通模式/prepare recovery/set consistency/node recovery
- 普通模式
	- 正常工作模式下，以PGCL为water line进行空洞填充
- Prepare Recovery模式
	- 尝试日志补齐阶段，primary节点会传递"consistency_term/[consistency term的最小起始位置, 最大NWL]"信息	
- Set Consistency模式
	- 最终日志补齐阶段，primary节点会传递"consistency_term/最终日志补齐点“信息
- Node Recovery模式
	- 此模式下，primary节点直接发送“最终日志补齐点”于本模块，用于对单个节点加入集群进行处理
	  
###### 总体功能流图

![绘图](./attachments/1629356917623.drawio.svg)


### 空洞填充
#### 请求状态列表
- 请求状态记录了gap请求的相关信息，用于对gap请求进行状态控制

### 空洞发现与空洞数据请求
- 空洞发现
	- 从term信息表取得target_lsn以下的空洞列表；
	- 顶部空洞：[NWL, target_lsn) 
- 空洞数据请求
 	- 为做到peer负载均衡，每个gap第一次进行空洞数据请求，以round robin形式，依次选择下一个peer为起点。
	- 获取数据: 向上述被选中的peer发送获取空洞数据请求(参数：空洞区间)后，更新请求状态列表
	- 不重复请求同一空洞区间
	- 大空洞会分割为多个标准空洞后，再进行发送

### 请求状态检测与处理

![绘图](./attachments/1629432112010.drawio.svg)

- 某个请求超时 - 继续向下一个peer请求数据
- 若已向所有peer发送过请求，则不再请求，返回失败
- 若收到请求回复为成功，在清理掉请求列表中相应条目

### 数据一致性处理
- prepare recovery/set consistency/node recovery模式下， 满足以下条件，对数据进行截断处理：
	- curr_term < consistency_term
	- curr_term_end_lsn < curr_term_end_lsn_other_node
- 完成set consistency recovery后，将consistency point后的所有wal进行截断

# 实现
### 重要数据结构

```
// 请求状态及列表
typedef struct request_status
{
	uint64 req_id;
	TimestampTz req_time;
	peer_idx pidx;
	peer_idx head_pidx;
	WalRange range;
} request_status;

List* request_status_list;
```
```
// 工作模式
typedef enum
{
	normal,
	prepare_recovery,
	final_recovery,
	node_recovery,
} mode;

```
### 重要函数
```
// 收到请求结果
static void do_handle_gap_fill_result(knl_thread_context* ctx, const gap_fill_result_option* option);
```

```
// prepare recovery处理
static void do_prepare_recovery(knl_thread_context* ctx, recovery_prepare_option* option);
```
```
// set consistency recovery处理
static void do_set_consistency_recovery(knl_thread_context* ctx, recovery_set_consistency_option* option);
```
```
// 请求状态列表维护
static void do_check_requests_status(knl_thread_context* ctx);
```
```
// gap发现与请求
static void gap_discovery_and_request(knl_thread_context * ctx, XLogRecPtr water_line);
```
```
// gap发现
static void compute_gaps_for_request(knl_thread_context * ctx, List ** request_gaps, XLogRecPtr water_line);
```
