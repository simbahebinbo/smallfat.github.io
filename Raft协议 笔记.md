---
title: Raft协议 笔记
tags: 
renderNumberedHeading: true
grammar_cjkRuby: true
---
# 解决了什么问题

# 有何特点

# 状态与规则定义
### 状态
###### Persistent State On All Servers
此处的persistent是指这些状态是持久化在storages上的
- currentTerm - lastest term the server has seen
- votedFor - the candidate id that received vote in current term(指当前term，该server投票给了哪个candidate，可能为null)
- log[] - log entries. Each entry contains command for state machine, and term that when the entry received by leader. (term是指所属entry被leader接收的term时间) 


###### Volatile State On All Servers
此处的Volatile是指下面的这些状态没有做持久化。
- commitIndex - index of highest log entry known to be commited (已经commit了的最新的log entry的编号)
- lastApplied - index of highest log entry which was applied to state machine

###### Volatile State On Leaders
- nextIndex[] - for each server, index of the next log entry to send to that server (这是指在leader服务器上，在leader向follower同步的过程中，待同步的下一个log entry index。 初始值是leader的最新的log_entry_index+1)
- matchIndex[] - for each server, index of highest log entry known to be replicated on that server(这是指在leader服务器上，leader已经向follower同步的最高的log entry index)

### Rules For Servers
###### for all servers
- if commitIndex > lastApplied, apply the entry log[lastApplied] to state machine, and increase lastApplied(这里只讲述了commitIndex与apply to state machine的关系。commitIndex变更的时机，另处会讲)
- if request or response contains term T > currentTerm, set currentTerm = T
	- 这条rule的目的是让server及时更新为最新的term


###### for followers
- responsed to RPCs from candidates and leaders (主要包括appendEntries RPC 和 RequestVotes RPC)
- if election timeout elapse without receiving appendEntries from current leader or without granting vote to candidates, then convert to candidate (这里讲了follower转为candidate的一个场景：在选举有效期(或者某段时间)内，follower没收到来自leader的appendEntries RPC，或者投票给任何candidate，则转换为candidate。)
	- 问题1：这里的election timeout是指谁的election timeout?还是指follower从初始化至成为candidate的那段时间，而不是选举超时时间？我觉得应该是指第二种.
	- 问题2：是选举有效期内还是选举超时后，才发生指定动作？

###### for candidates
- while starting election
	- increase currentTerm
	- Vote for self
	- reset election timeout to a random time
	- send RequestVote RPC to all other servers
- become leader - received vote from majority of servers
	- 这里servers的数量，应该是系统启动时的servers的数量。也就是说，raft并不支持系统运行过程中的简单动态加入，否则算法会出问题。
	
- become follower: received appendEntries from new Leader
- if election timeout elaspse, restart a new election
	- 这会使得其他的candidate的term都失效，从而提出一个新的问题：收到带有更新term的RequestVote RPC，该怎么处理？

###### for leaders
- Upon election: send empty appendEntries to all other servers as heartbeat; and must repeat it during idle periods to prevent election timeout
	-  重复发送empty appendEntries RPC是为了防止follower转为candidate，请见"rules for followers"第二条
- if command received from client: append entry to local log, respond after entry applied to state machine
- if last_log_index>=nextIndex for a follower, then send appendEntries with log entries starting at nextIndex to the follower
	- 问题： log entries starting at nextIndex，这里nextIndex换成matchIndex是不是更合适？因为matchIndex记录的是已经匹配的index
	- if success, update nextIndex and matchIndex
	- if failed, retry with nextIndex--

# 业务模块
### 状态变迁
![raft state transfer](https://gitee.com/string_coder/xiaoshujiang/raw/master/raft-state-transfer.png)

如上图，raft中有三个角色：
- Follower - 普通群众
- Candidate - 候选人
- Leader - 领导

互相之间的状态变迁定义如下（这与上一节中状态定义是对应的）：
 - Startup -> Follower - 系统初始状态下，所有节点都是Follower
 - Follower -> Candidate - 每个Follower有一个静默时间t。若t时间内，Follower没有收到来自Leader的任何消息（heartbeat等），则Follower转变为Candidate，开始新的选举term
 - Candidate -> Follower - 当1.发现了新的leader已经选出 或者 2. 发现了新的(更高的？？)term 时，发生此状态改变
 - Candidate -> Leader - 若收到了(n/2) + 1个vote时，成为Leader
 - Candidate -> Candidate - 当前选举term内，没有收到多数vote，且term超时时间到，则开始一个新的term
 - Leader -> Follower - 若发现了更高的term，则转为Follower

### 任务Term
Term在这里的概念相当于“一轮”的意思。在每个Term里，包含“选举期”和“工作期”两部分(也有可能没有工作期因为选举没有成功或者被中断)。如下图：

![raft term](https://gitee.com/string_coder/xiaoshujiang/raw/master/raft-term.png)

以下几点是我个人的理解：
- Term是一个逻辑上的时间概念，每个node即使在同一个Term，有可能在时钟时间上，是有误差的。
- 每个通信包里面都包含着Term信息
- 每个node会拒绝接受旧的Term的通信包
- 每个node会更新到通信包中的最新Term

### Leader Election
- 每个node在一次Term里面只能投一次票
- 取得多数votes的node成为leader,即: votes >= (n/2) + 1
- 每个candidate的选举超时时间是随机时间，目的是防止split votes现象

### Log Replication
