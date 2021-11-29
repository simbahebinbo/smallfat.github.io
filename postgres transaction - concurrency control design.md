---
title: postgres transaction - concurrency control design
tags: 
grammar_cjkRuby: true
---
# transaction id
- transaction id (txid), is assigned by the transaction manager
- PostgreSQL's txid is a 32-bit unsigned integer

# Tuple Struct
- A heap tuple comprises three parts, i.e. the HeapTupleHeaderData structure, NULL bitmap, and user data 
### HeapTupleHeadData
-  HeapTupleHeaderData structure contains seven fields, four fields are required in the subsequent sections.
-  t_xmin holds the txid of the transaction that inserted this tuple.
-  t_xmax holds the txid of the transaction that deleted or updated this tuple
    -  If this tuple has not been deleted or updated, t_xmax is set to 0, which means INVALID.
-  t_cid holds the command id (cid), which means how many SQL commands were executed before this command was executed within the current transaction beginning from 0
-  t_ctid holds the tuple identifier (tid) that points to itself or a new tuple. tid, described in Section 1.3, is used to identify a tuple within a table. When this tuple is updated, the t_ctid of this tuple points to the new tuple; otherwise, the t_ctid points to itself.

### Operating Tuple
###### Insert
![image](https://www.interdb.jp/pg/img/fig-5-04.png)

- t_xmin is set to 99 because this tuple is inserted by txid 99.
- t_xmax is set to 0 because this tuple has not been deleted or updated.
- t_cid is set to 0 because this tuple is the first tuple inserted by txid 99.
- t_ctid is set to (0,1), which points to itself, because this is the latest tuple.

###### Delete
![image](https://www.interdb.jp/pg/img/fig-5-05.png)

- If txid 111 is committed, Tuple_1 is no longer required. Generally, unneeded tuples are referred to as dead tuples in PostgreSQL.
- Dead tuples should eventually be removed from pages. Cleaning dead tuples is referred to as VACUUM processing

###### Update
![image](https://www.interdb.jp/pg/img/fig-5-06.png)

- When the first UPDATE command is executed, Tuple_1 is logically deleted by setting txid 100 to the t_xmax, and then Tuple_2 is inserted. 
- Then, the t_ctid of Tuple_1 is rewritten to point to Tuple_2. The header fields of both Tuple_1 and Tuple_2 are as follows.

- When the second UPDATE command is executed, as in the first UPDATE command, Tuple_2 is logically deleted and Tuple_3 is inserted. 

# Commit Log
### overview
- PostgreSQL holds the statuses of transactions in the Commit Log. 
- The Commit Log, often called the clog, is allocated to the shared memory, 
- used throughout transaction processing.

### Transaction Status
-  IN_PROGRESS, COMMITTED, ABORTED, and SUB_COMMITTED

### Principle
![image](https://www.interdb.jp/pg/img/fig-5-07.png)

- The clog comprises one or more 8 KB pages in shared memory. 
- The clog logically forms an array. 
- The indices of the array correspond to the respective transaction ids
- each item in the array holds the status of the corresponding transaction id
- When the current txid advances and the clog can no longer store it, a new page is appended???

### Maintenance
- PostgreSQL shuts down or whenever the checkpoint process runs, the data of the clog are written into files stored under the pg_clog subdirectory(pg_xact after v10)
- When PostgreSQL starts up, the data stored in the pg_clog's files (pg_xact's files) are loaded to initialize the clog.

# Transaction Snapshot
- a dataset that stored information about whether all transactions are active. ( an active transaction means it is in progress or has not yet started.)

### format and meaning

```
testdb=# SELECT txid_current_snapshot();
 txid_current_snapshot 
-----------------------
 100:104:100,102
(1 row)
```

![image](https://www.interdb.jp/pg/img/fig-5-08.png)

### Transaction Isolation Levels

Isolation Level| 脏读 | 更新丢失 | 不可重复读| 幻读
---|---|---|---|---
Read Uncommitted	读未提交 | 可能发生 | 可能发生 | 可能发生 | 可能发生
Read Committed	读已提交| 不会发生 | 可能发生 | 可能发生 | 可能发生
Repeatable Read	可重复读| 不会发生 | 不会发生 | 不会发生 | 可能发生
Read Committed	可串行化| 不会发生 | 不会发生 | 不会发生 | 不会发生


### Isolation and transaction snapshot
- Transaction snapshots are provided by the transaction manager. 
- In the READ COMMITTED isolation level, the transaction obtains a snapshot whenever an SQL command is executed; 
- otherwise (REPEATABLE READ or SERIALIZABLE), the transaction only gets a snapshot when the first SQL command is executed. 
- The obtained transaction snapshot is used for a visibility check of tuples

# Visibility Check Rules
```
Rule 1: If Status(t_xmin) = ABORTED ⇒ Invisible
Rule 2: If Status(t_xmin) = IN_PROGRESS ∧ t_xmin = current_txid ∧ t_xmax = INVAILD ⇒ Visible
Rule 3: If Status(t_xmin) = IN_PROGRESS ∧ t_xmin = current_txid ∧ t_xmax ≠ INVAILD ⇒ Invisible
Rule 4: If Status(t_xmin) = IN_PROGRESS ∧ t_xmin ≠ current_txid ⇒ Invisible
Rule 5: If Status(t_xmin) = COMMITTED ∧ Snapshot(t_xmin) = active ⇒ Invisible
Rule 6: If Status(t_xmin) = COMMITTED ∧ (t_xmax = INVALID ∨ Status(t_xmax) = ABORTED) ⇒ Visible
Rule 7: If Status(t_xmin) = COMMITTED ∧ Status(t_xmax) = IN_PROGRESS ∧ t_xmax = current_txid ⇒ Invisible
Rule 8: If Status(t_xmin) = COMMITTED ∧ Status(t_xmax) = IN_PROGRESS ∧ t_xmax ≠ current_txid ⇒ Visible
Rule 9: If Status(t_xmin) = COMMITTED ∧ Status(t_xmax) = COMMITTED ∧ Snapshot(t_xmax) = active ⇒ Visible
Rule 10: If Status(t_xmin) = COMMITTED ∧ Status(t_xmax) = COMMITTED ∧ Snapshot(t_xmax) ≠ active ⇒ Invisible

```


# Visibility Check
- how heap tuples of the appropriate versions in a given transaction are selected
- how PostgreSQL prevents the anomalies: Dirty Reads, Repeatable Reads and Phantom Reads.

# Searialization Snapshot Isolation
