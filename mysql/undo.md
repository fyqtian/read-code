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