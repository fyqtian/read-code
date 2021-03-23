http://mysql.taobao.org/monthly/2015/05/01/

https://www.cnblogs.com/f-ck-need-u/archive/2018/05/08/9010872.html



<img src="..\images\2e5bff4910ec189fe1ee6e2ecc7b4bbe.png" alt="2e5bff4910ec189fe1ee6e2ecc7b4bbe" style="zoom: 33%;" />

将redo log的写入拆成了两个步骤：prepare和commit，**这就是"两阶段提交"。**



由于redo log和binlog是**两个独立的逻辑**，如果不用两阶段提交，要么就是先写完redo log再写binlog，或者采用反过来的顺序。我们看看这两种方式会有什么问题。

仍然用前面的update语句来做例子。假设当前ID=2的行，字段c的值是0，再假设执行update语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了crash，会出现什么情况呢？

1. **先写redo log后写binlog**。假设在redo log写完，binlog还没有写完的时候，MySQL进程异常重启。由于我们前面说过的，redo log写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行c的值是1。
   但是由于binlog没写完就crash了，这时候binlog里面就没有记录这个语句。因此，之后备份日志的时候，存起来的binlog里面就没有这条语句。
   然后你会发现，如果需要用这个binlog来恢复临时库的话，由于这个语句的binlog丢失，这个临时库就会少了这一次更新，恢复出来的这一行c的值就是0，与原库的值不同。
2. **先写binlog后写redo log**。如果在binlog写完之后crash，由于redo log还没写，崩溃恢复以后这个事务无效，所以这一行c的值是0。但是binlog里面已经记录了“把c从0改成1”这个日志。所以，在之后用binlog来恢复的时候就多了一个事务出来，恢复出来的这一行c的值就是1，与原库的值不同。

可以看到，如果不使用“两阶段提交”，那么数据库的状态就有可能和用它的日志恢复出来的库的状态不一致。

简单说，redo log和binlog都可以用于表示事务的提交状态，**而两阶段提交就是让这两个状态保持逻辑上的一致**。

redo log用于**保证crash-safe能力**。innodb_flush_log_at_trx_commit这个参数设置成1的时候，表示每次事务的redo log都直接持久化到磁盘。这个参数我建议你设置成1，这样可以保证MySQL异常重启之后数据不丢失。

sync_binlog这个参数设置成1的时候，表示每次事务的binlog都持久化到磁盘。这个参数我也建议你设置成1，这样可以保证MySQL异常重启之后binlog不丢失。









![2338659816-5c3c5e91b7290_fix732](..\images\2338659816-5c3c5e91b7290_fix732.png)

### Redo 的作用

Redo log的主要作用是用于数据库的**崩溃恢复**

### Redo 的组成

Redo log可以简单分为以下两个部分：

- 一是内存中重做日志缓冲 (redo log buffer),是易失的，在内存中
- 二是重做日志文件 (redo log file)，是持久的，保存在磁盘中

### 什么时候写Redo?

上面那张图简单地体现了Redo的写入流程，这里再细说下写入Redo的时机：

- 在数据页修改完成之后，在脏页刷出磁盘之前，写入redo日志。**注意的是先修改数据，后写日志**

- **redo日志比数据页先写回磁盘**

- 聚集索引、二级索引、undo页面的修改，均需要记录Redo日志。

  

**宏观上把握redo log 流转过程**

- 第一步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝
- 第二步：生成一条重做日志并写入redo log buffer，记录的是数据被修改后的值
- 第三步：当事务commit时，将redo log buffer中的内容刷新到 redo log file，对 redo log file采用追加写的方式
- 第四步：定期将内存中修改的数据刷新到磁盘中



### redo如何保证 事务的持久性？

InnoDB是事务的存储引擎，其通过**Force Log at Commit 机制**实现事务的持久性，即当事务提交时，先将 redo log buffer **写入到 redo log file 进行持久化**，待事务的commit操作完成时才算完成。这种做法也被称为 **Write-Ahead Log(预先日志持久化)**，在持久化一个数据页之前，先将内存中相应的日志页持久化。

为了保证每次日志都写入redo log file，在每次将redo buffer写入redo log file之后，默认情况下，InnoDB存储引擎都需要调用一次 **fsync操作**,因为重做日志打开并没有 O_DIRECT选项，所以重做日志先写入到文件系统缓存。为了确保重做日志写入到磁盘，必须进行一次 fsync操作。fsync是一种系统调用操作，其fsync的效率取决于磁盘的性能，因此磁盘的性能也影响了事务提交的性能，也就是数据库的性能。
**(O_DIRECT选项是在Linux系统中的选项，使用该选项后，对文件进行直接IO操作，不经过文件系统缓存，直接写入磁盘)**



上面提到的**Force Log at Commit机制**就是靠InnoDB存储引擎提供的参数 `innodb_flush_log_at_trx_commit`来控制的，该参数可以控制 redo log刷新到磁盘的策略，设置该参数值也可以允许用户设置非持久性的情况发生，具体如下：

- 当设置参数为1时，（默认为1），表示事务提交时必须调用一次 `fsync` 操作，最安全的配置，保障持久性
- 当设置参数为2时，则在事务提交时只做 **write** 操作，只保证将redo log buffer写到系统的页面缓存中，不进行fsync操作，因此如果MySQL数据库宕机时 不会丢失事务，但操作系统宕机则可能丢失事务
- 当设置参数为0时，表示事务提交时不进行写入redo log操作，这个操作仅在master thread 中完成，而在master thread中每1秒进行一次重做日志的fsync操作，因此实例 crash 最多丢失1秒钟内的事务。（master thread是负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性）



`sync`和`write`操作实际上是系统调用函数，在很多持久化场景都有使用到，比如 Redis 的AOF持久化中也使用到两个函数。`fsync`操作 将数据提交到硬盘中，强制硬盘同步，将一直阻塞到写入硬盘完成后返回，大量进行`fsync`操作就有性能瓶颈，而`write`操作将数据写到系统的页面缓存后立即返回，后面依靠系统的调度机制将缓存数据刷到磁盘中去,其顺序是user buffer——> page cache——>disk。



**除了上面谈到的Force Log at Commit机制保证事务的持久性，实际上重做日志的实现还要依赖于mini-transaction。**

### Redo在InnoDB中是如何实现的？与mini-transaction的联系？

Redo的实现实则跟mini-transaction紧密相关，mini-transaction是一种InnoDB内部使用的机制，通过mini-transaction来**保证并发事务操作下以及数据库异常时数据页中数据的一致性**，但它不属于事务。

**为了使得mini-transaction保证数据页数据的一致性，mini-transaction必须遵循以下三种协议**：

- The FIX Rules
- Write-Ahead Log
- Force-log-at-commit

**The FIX Rules**

修改一个数据页时需要获得该页的x-latch(排他锁)，获取一个数据页时需要该页的s-latch(读锁或者称为共享锁) 或者是 x-latch，持有该页的锁直到修改或访问该页的操作完成。

**Write-Ahead Log**

在前面阐述中就提到了Write-Ahead Log(预先写日志)。在持久化一个数据页之前，必须先将内存中相应的日志页持久化。每个页都有一个LSN(log sequence number)，代表日志序列号，（LSN占用8字节，单调递增), 当一个数据页需要写入到持久化设备之前，要求内存中小于该页LSN的日志先写入持久化设备

那为什么必须要先写日志呢？可不可以不写日志，直接将数据写入磁盘？原则上是可以的，只不过会产生一些问题，数据修改会产生随机IO，但日志是顺序IO，append方式顺序写，是一种串行的方式，这样才能充分利用磁盘的性能。

**Force-log-at-commit**

这一点也就是前文提到的如何保证事务的持久性的内容，这里再次总结一下，与上面的内容相呼应。在一个事务中可以修改多个页，Write-Ahead Log 可以保证单个数据页的一致性，但是无法保证事务的持久性，Force-log-at-commit 要求当一个事务提交时，其产生所有的mini-transaction 日志必须刷新到磁盘中，若日志刷新完成后，在缓冲池中的页刷新到持久化存储设备前数据库发生了宕机，那么数据库重启时，可以通过日志来保证数据的完整性。



<img src="..\images\1470992598-5c3c5e91c5251_fix732.png" alt="1470992598-5c3c5e91c5251_fix732" style="zoom:75%;" />

上图表示了重做日志的写入流程，每个mini-transaction对应每**一条DML操作**，比如一条update语句，其由一个mini-transaction来保证，对数据修改后，产生redo1，首先将其写入mini-transaction私有的Buffer中，update语句结束后，将redo1从私有Buffer拷贝到公有的Log Buffer中。当整个外部事务提交时，将redo log buffer再刷入到redo log file中。





































1.redo log**通常是物理日志**，记录的是数据页的**物理修改**，而不是某一行或某几行修改成怎样怎样，它用来恢复提交后的物理数据页(恢复数据页，且只能恢复到最后一次提交的位置)。
2.undo用来**回滚行记录到某个版本**。undo log一般是**逻辑日志，根据每行记录进行记录**。



redo log不是二进制日志。虽然二进制日志中也记录了innodb表的很多操作，**也能实现重做的功能，**但是它们之间有很大区别。



1. 二进制日志是在**存储引擎的上层**产生的，不管是什么存储引擎，对数据库进行了修改都会产生二进制日志。而redo log是innodb层产生的，只记录该存储引擎中表的修改。并且**二进制日志先于redo log被记录**。具体的见后文group commit小结。
2. 二进制日志记录操作的方法是逻辑性的语句。即便它是基于行格式的记录方式，其本质也还是逻辑的SQL设置，如该行记录的每列的值是多少。而redo log是在物理格式上的日志，它记录的是数据库中每个页的修改。
3. 二进制日志只在**每次事务提交的时候**一次性写入缓存中的日志"文件"(对于非事务表的操作，则是每次执行语句成功后就直接写入)。而redo log**在数据准备修改前写入缓存中的redo log中**，然后才对缓存中的数据执行修改操作；而且保证在发出事务提交指令时，先向缓存中的redo log写入日志，写入完成后才执行提交动作。
4. 因为二进制日志只在提交的时候一次性写入，所以二进制日志中的记录方式和提交顺序有关，且一次提交对应一次记录。而redo log中是记录的物理页的修改，redo log文件中同一个事务可能多次记录，最后一个提交的事务记录会覆盖所有未提交的事务记录。例如事务T1，可能在redo log中记录了 T1-1,T1-2,T1-3，T1* 共4个操作，其中 T1* 表示最后提交时的日志记录，所以对应的数据页最终状态是 T1* 对应的操作结果。而且redo log是并发写入的，不同事务之间的不同版本的记录会穿插写入到redo log文件中，例如可能redo log的记录方式如下： T1-1,T1-2,T2-1,T2-2,T2*,T1-3,T1* 
5. 事务日志记录的是物理页的情况，**它具有幂等性**，因此记录日志的方式极其简练。幂等性的意思是多次操作前后状态是一样的，例如新插入一行后又删除该行，前后状态没有变化。而二进制日志记录的是所有影响数据的操作，记录的内容较多。例如插入一行记录一次，删除该行又记录一次。





MySQL支持用户自定义在commit时如何将log buffer中的日志刷log file中。这种控制通过变量 innodb_flush_log_at_trx_commit 的值来决定。该变量有3种值：0、1、2，默认为1。但注意，这个变量只是控制commit动作是否刷新log buffer到磁盘。

- 当设置为1的时候，事务每次提交都会将log buffer中的日志写入os buffer并调用fsync()刷到log file on disk中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。
- 当设置为0的时候，**事务提交时不会将log buffer中日志写入到os buffer**，而是每秒写入os buffer并调用fsync()写入到log file on disk中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。
- 当设置为2的时候，每次提交都仅写入到os buffer，然后是每秒调用fsync()将os buffer中的日志写入到log file on disk。

0：log buffer将每秒一次地写入log file中，并且log file的flush(刷到磁盘)操作同时进行。该模式下在事务提交的时候，不会主动触发写入磁盘的操作。

1：每次事务提交时MySQL都会把log buffer的数据写入log file，并且flush(刷到磁盘)中去，该模式为系统默认。

2：每次事务提交时MySQL都会把log buffer的数据写入log file，但是flush(刷到磁盘)操作并不会同时进行。该模式下，MySQL会每秒执行一次 flush(刷到磁盘)操作。



![24700673-178589340b50f573](../images/733013-20180508104623183-690986409.png)



## 1.3 日志块(log block)

innodb存储引擎中，redo log**以块为单位**进行存储的，**每个块占512字节**，这称为redo log block。所以不管是log buffer中还是os buffer中以及redo log file on disk中，都是这样以512字节的块存储的。

每个redo log block由3部分组成：**日志块头、日志块尾和日志主体**。其中日志块头占用12字节，日志块尾占用8字节，所以每个redo log block的日志主体部分只有512-12-8=492字节。



##  1.9 innodb的恢复行为

在启动innodb的时候，不管上次是**正常关闭还是异常关闭**，总是会进行恢复操作。

因为redo log记录的是数据页的物理变化，因此恢复的时候速度比逻辑日志(如二进制日志)要快很多。而且，innodb自身也做了一定程度的优化，让恢复速度变得更快。

重启innodb时，checkpoint表示已经完整刷到磁盘上data page上的LSN，因此恢复时仅需要恢复从checkpoint开始的日志部分。例如，**当数据库在上一次checkpoint的LSN为10000时宕机**，且事务是已经提交过的状态。启动数据库时会检查磁盘中数据页的LSN，如果数据页的LSN小于日志中的LSN，则会从检查点开始恢复。

还有一种情况，在宕机前正处于checkpoint的刷盘过程，且数据页的刷盘进度超过了日志页的刷盘进度。这时候一宕机，数据页中记录的LSN就会大于日志页中的LSN，在重启的恢复过程中会检查到这一情况，这时超出日志进度的部分将不会重做，因为这本身就表示已经做过的事情，无需再重做。

另外，事务日志具有幂等性，所以多次操作得到同一结果的行为在日志中只记录一次。而二进制日志不具有幂等性，多次操作会全部记录下来，在恢复的时候会多次执行二进制日志中的记录，速度就慢得多。例如，某记录中id初始值为2，通过update将值设置为了3，后来又设置成了2，在事务日志中记录的将是无变化的页，根本无需恢复；而二进制会记录下两次update操作，恢复时也将执行这两次update操作，速度比事务日志恢复更慢。



## 1.10 和redo log有关的几个变量

- innodb_flush_log_at_trx_commit={0|1|2} # 指定何时将事务日志刷到磁盘，默认为1。
  - 0表示每秒将"log buffer"同步到"os buffer"且从"os buffer"刷到磁盘日志文件中。
  - 1表示每事务提交都将"log buffer"同步到"os buffer"且从"os buffer"刷到磁盘日志文件中。
  - 2表示每事务提交都将"log buffer"同步到"os buffer"但每秒才从"os buffer"刷到磁盘日志文件中。
- innodb_log_buffer_size：# log buffer的大小，默认8M
- innodb_log_file_size：#事务日志的大小，默认5M
- innodb_log_files_group =2：# 事务日志组中的事务日志文件个数，默认2个
- innodb_log_group_home_dir =./：# 事务日志组路径，当前目录表示数据目录
- innodb_mirrored_log_groups =1：# 指定事务日志组的镜像组个数，但镜像功能好像是强制关闭的，所以只有一个log group。在MySQL5.7中该变量已经移除。