### undo

http://mysql.taobao.org/monthly/2015/04/01/



Undo log是InnoDB MVCC事务特性的重要组成部分。**当我们对记录做了变更操作时就会产生undo记录**，Undo记录默认被记录到系统表空间(ibdata)中，但从5.6开始，也可以使用独立的Undo 表空间。

**Undo记录中存储的是老版本数据**，当一个旧的事务需要读取数据时，**为了能读取到老版本的数据，需要顺着undo链找到满足其可见性的记录**。当版本链很长时，通常可以认为这是个比较耗时的操作。

大多数对数据的变更操作包括INSERT/DELETE/UPDATE，**其中INSERT操作在事务提交前只对当前事务可见**，因此产生的Undo日志可以在事务提交后直接删除（谁会对刚插入的数据有可见性需求呢！！），**而对于UPDATE/DELETE则需要维护多版本信息，在InnoDB里，UPDATE和DELETE操作产生的Undo日志被归成一类，即update_undo。**



### 基本文件结构

为了保证事务并发操作时，在写各自的undo log时不产生冲突，InnoDB采用回滚段的方式来维护undo log的并发写入和持久化。回滚段实际上是一种 Undo 文件组织方式，每个回滚段又有多个undo log slot。具体的文件组织方式如下图所示：

![01](..\images\01.png)

图展示了基本的Undo回滚段布局结构，其中:

1. rseg0预留在系统表空间ibdata中;
2. rseg 1~rseg 32这32个回滚段存放于临时表的系统表空间中;
3. rseg33~ 则根据配置存放到独立undo表空间中（如果没有打开独立Undo表空间，则存放于ibdata中）





**如果我们使用独立Undo tablespace**，则总是从第一个Undo space开始轮询分配undo 回滚段。大多数情况下这是OK的，但假设我们将回滚段的个数从33开始依次递增配置到128，就可能导致所有的回滚段都存放在同一个undo space中。(参考函数trx_sys_create_rsegs 以及 [bug#74471](http://bugs.mysql.com/bug.php?id=74471))

每个回滚段维护了一个段头页，在该page中又划分了1024个slot(TRX_RSEG_N_SLOTS)，每个slot又对应到一个undo log对象，因此理论上InnoDB最多支持 96 * 1024个普通事务。







#### 多版本控制

InnoDB的多版本使用undo来构建，这很好理解，**undo记录中包含了记录更改前的镜像**，如果更改数据的**事务未提交**，**对于隔离级别大于等于read commit的事务而言，它不应该看到已修改的数据**，**而是应该给它返回老版本的数据**。



由于在修改聚集索引记录时，总是存储了回滚段指针和事务id，**可以通过该指针找到对应的undo 记录**，通过事务Id来判断记录的可见性。当旧版本记录中的事务id对当前事务而言是不可见时，则继续向前构建，直到找到一个可见的记录或者到达版本链尾部。（关于事务可见性及read view，可以参阅我们[之前的月报](http://mysql.taobao.org/index.php?title=MySQL内核月报_2014.12#MySQL.C2.B7.E3.80.80.E6.80.A7.E8.83.BD.E4.BC.98.E5.8C.96.C2.B75.7_Innodb.E4.BA.8B.E5.8A.A1.E7.B3.BB.E7.BB.9F)）







## 2.1 基本概念

undo log有两个作用：提供回滚和多个行版本控制(MVCC)。

在数据修改的时候，**不仅记录了redo，还记录了相对应的undo**，如果因为某些原因导致事务失败或回滚了，可以借助该undo进行回滚。

undo log和redo log记录物理日志不一样，**它是逻辑日志**。**可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。**



当执行rollback时，就可以从undo log中的**逻辑记录读取到相应的内容并进行回滚**。有时候应用到行版本控制的时候，也是通过undo log来实现的：当读取的某一行被其他事务锁定时，它可以从undo log中分析出该行记录以前的数据是什么，从而提供该行版本信息，让用户实现非锁定一致性读取。

undo log是采用段(segment)的方式来记录的，每个undo操作在记录的时候占用一个undo log segment

另外，undo log也会产生redo log，因为undo log也要实现持久性保护。







# binlog和事务日志的先后顺序及group commit

为了提高性能，通常会将有关联性的多个数据修改操作放在一个事务中，这样可以避免对每个修改操作都执行完整的持久化操作。这种方式，可以看作是人为的组提交(group commit)。

除了将多个操作组合在一个事务中，记录binlog的操作也可以按组的思想进行优化：将多个事务涉及到的binlog一次性flush，而不是每次flush一个binlog。

事务在提交的时候不仅会记录事务日志，还会记录二进制日志，但是它们谁先记录呢？二进制日志是MySQL的上层日志，先于存储引擎的事务日志被写入。

在MySQL5.6以前，当事务提交(即发出commit指令)后，MySQL接收到该信号进入commit prepare阶段；进入prepare阶段后，立即写内存中的二进制日志，写完内存中的二进制日志后就相当于确定了commit操作；然后开始写内存中的事务日志；最后将二进制日志和事务日志刷盘，它们如何刷盘，分别由变量 sync_binlog 和 innodb_flush_log_at_trx_commit 控制。

但因为要保证二进制日志和事务日志的一致性，在提交后的prepare阶段会启用一个**prepare_commit_mutex**锁来保证它们的顺序性和一致性。但这样会导致开启二进制日志后group commmit失效，特别是在主从复制结构中，几乎都会开启二进制日志。

在MySQL5.6中进行了改进。提交事务时，在存储引擎层的上一层结构中会将事务按序放入一个队列，队列中的第一个事务称为leader，其他事务称为follower，leader控制着follower的行为。虽然顺序还是一样先刷二进制，再刷事务日志，但是机制完全改变了：删除了原来的prepare_commit_mutex行为，也能保证即使开启了二进制日志，group commit也是有效的。