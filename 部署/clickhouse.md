



https://blog.csdn.net/qq_37933018/article/details/111309778

同步kafka

https://www.jianshu.com/p/745531f7650e

配置文件 

/etc/clickhouse-server/config.xml

/etc/clickhouse-server/metrika.xml 集群配置、ZK配置、分片配置等

端口默认9000

clickhouse-client -h host --port 9000

--user -u 默认default

--password 默认空

--query -q 

--database -d

--multiline -m 允许多行语句查询

-- time -t 

--format -f

--config-file

--stacktrace





clickhouse-report  打印运行时参数

### 集群

[Clickhouse分布式集群搭建 - Suncle's Blog](https://suncle.me/2019/07/27/clickhouse-build-distributed-cluster/)



### **数据类型**

https://vkingnew.blog.csdn.net/article/details/106644698 



### 运算操作

https://vkingnew.blog.csdn.net/article/details/106651966



### 多磁盘挂载

https://vkingnew.blog.csdn.net/article/details/106748472



### ttl

https://vkingnew.blog.csdn.net/article/details/106772513

https://altinity.com/blog/2020/3/23/putting-things-where-they-belong-using-new-ttl-moves



### list view

https://vkingnew.blog.csdn.net/article/details/106774906

```
select name ,value,changed,min,max,readonly,type from system.settings where name like '%live_view%';

Live view 目前还是实验功能（experimental feature）需要开启参数allow_experimental_live_view ：
手动设置：


SET allow_experimental_live_view = 1;


create table t_lv(id UInt64)engine=Log;

create live view lv_count as select count(1) from t_lv;

watch lv_count;

insert into t_lv select rand() from numbers(10);
```



### materialized view

https://vkingnew.blog.csdn.net/article/details/106775064

```
CREATE [MATERIALIZED] VIEW [IF NOT EXISTS] [db.]table_name [TO[db.]name] [ENGINE = engine] [POPULATE] AS SELECT ..

创建雾化视图的限制：
1.必须指定物化视图的engine 用于数据存储
2.TO [db].[table]语法的时候，不得使用POPULATE。
3.查询语句(select）可以包含下面的子句： DISTINCT, GROUP BY, ORDER BY, LIMIT…
4.雾化视图的alter操作有些限制，操作起来不大方便。
5.若物化视图的定义使用了TO [db.]name 子语句，则可以将目标表的视图 卸载 DETACH 在装载 ATTACH 
 
物化视图的数据更新:
1.物化视图创建好之后，若源表被写入新数据则物化视图也会同步更新
2.POPULATE 关键字决定了物化视图的更新策略：
  若有POPULATE 则在创建视图的过程会将源表已经存在的数据一并导入，类似于 create table ... as 
  若无POPULATE 则物化视图在创建之后没有数据，只会在创建只有同步之后写入源表的数据.
clickhouse 官方并不推荐使用populated，因为在创建物化视图的过程中同时写入的数据不能被插入物化视图。
3.物化视图不支持同步删除，若源表的数据不存在（删除了）则物化视图的数据仍然保留
 
4.物化视图是野种特殊的数据表，可以用show tables 查看
5.物化视图数据的删除：

————————————————
版权声明：本文为CSDN博主「vkingnew」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/vkingnew/article/details/106775064
```





```xml
config


<?xml version="1.0"?>
<yandex>
   <!-- 日志 -->
   <logger>
       <level>trace</level>
       <log>/data1/clickhouse/log/server.log</log>
       <errorlog>/data1/clickhouse/log/error.log</errorlog>
       <size>1000M</size>
       <count>10</count>
   </logger>

   <!-- 端口 -->
   <http_port>8123</http_port>
   <tcp_port>9000</tcp_port>
   <interserver_http_port>9009</interserver_http_port>

   <!-- 本机域名 -->
   <interserver_http_host>这里需要用域名，如果后续用到复制的话</interserver_http_host>

   <!-- 监听IP -->
   <listen_host>0.0.0.0</listen_host>
   <!-- 最大连接数 -->
   <max_connections>64</max_connections>

   <!-- 没搞懂的参数 -->
   <keep_alive_timeout>3</keep_alive_timeout>

   <!-- 最大并发查询数 -->
   <max_concurrent_queries>16</max_concurrent_queries>

   <!-- 单位是B -->
   <uncompressed_cache_size>8589934592</uncompressed_cache_size>
   <mark_cache_size>10737418240</mark_cache_size>

   <!-- 存储路径 -->
   <path>/data1/clickhouse/</path>
   <tmp_path>/data1/clickhouse/tmp/</tmp_path>

   <!-- user配置 -->
   <users_config>users.xml</users_config>
   <default_profile>default</default_profile>

   <log_queries>1</log_queries>

   <default_database>default</default_database>

   <remote_servers incl="clickhouse_remote_servers" />
   <zookeeper incl="zookeeper-servers" optional="true" />
   <macros incl="macros" optional="true" />

   <!-- 没搞懂的参数 -->
   <builtin_dictionaries_reload_interval>3600</builtin_dictionaries_reload_interval>

   <!-- 控制大表的删除 -->
   <max_table_size_to_drop>0</max_table_size_to_drop>

   <include_from>/data1/clickhouse/metrika.xml</include_from>
</yandex>



----------------
metrika.xml

<yandex>
<!-- 集群配置 -->
<clickhouse_remote_servers>
	<!-- 集群名称-->
    <bip_ck_cluster>
        <shard>
            <internal_replication>false</internal_replication>
            <replica>
                <host>ck1.xxxx.com.cn</host>
                <port>9000</port>
                <user>default</user>
                <password>******</password>
            </replica>
			<replica>
                <host>ck2.xxxx.com.cn</host>
                <port>9000</port>
                <user>default</user>
                <password>******</password>
            </replica>
        </shard>
        <shard>
            <internal_replication>false</internal_replication>
            <replica>
                <host>ck2.xxxx.com.cn</host>
                <port>9000</port>
                <user>default</user>
                <password>******</password>
            </replica>
			<replica>
                <host>ck3.xxxxa.com.cn</host>
                <port>9000</port>
                <user>default</user>
                <password>******</password>
            </replica>
        </shard>
        <shard>
            <internal_replication>false</internal_replication>
            <replica>
                <host>ck3.xxxxa.com.cn</host>
                <port>9000</port>
                <user>default</user>
                <password>******</password>
            </replica>
			<replica>
                <host>ck1.xxxx.com.cn</host>
                <port>9000</port>
                <user>default</user>
                <password>******</password>
            </replica>
        </shard>
    </bip_ck_cluster>
</clickhouse_remote_servers>

<!-- 本节点副本名称（这里无用） -->
<macros>
    <replica>ck1</replica>
</macros>

<!-- 监听网络（貌似重复） -->
<networks>
   <ip>::/0</ip>
</networks>
<!-- ZK  -->
<zookeeper-servers>
  <node index="1">
    <host>1.xxxx.sina.com.cn</host>
    <port>2181</port>
  </node>
  <node index="2">
    <host>2.xxxx.sina.com.cn</host>
    <port>2181</port>
  </node>
  <node index="3">
    <host>3.xxxxp.sina.com.cn</host>
    <port>2181</port>
  </node>
</zookeeper-servers>
<!-- 数据压缩算法  -->
<clickhouse_compression>
<case>
  <min_part_size>10000000000</min_part_size>
  <min_part_size_ratio>0.01</min_part_size_ratio>
  <method>lz4</method>
</case>
</clickhouse_compression>

</yandex>


------
user
<?xml version="1.0"?>
<yandex>
    <profiles>
        <!-- 读写用户设置  -->
        <default>
            <max_memory_usage>10000000000</max_memory_usage>
            <use_uncompressed_cache>0</use_uncompressed_cache>
            <load_balancing>random</load_balancing>
        </default>

        <!-- 只写用户设置  -->
        <readonly>
            <max_memory_usage>10000000000</max_memory_usage>
            <use_uncompressed_cache>0</use_uncompressed_cache>
            <load_balancing>random</load_balancing>
            <readonly>1</readonly>
        </readonly>
    </profiles>

    <!-- 配额  -->
    <quotas>
        <!-- Name of quota. -->
        <default>
            <interval>
                <duration>3600</duration>
                <queries>0</queries>
                <errors>0</errors>
                <result_rows>0</result_rows>
                <read_rows>0</read_rows>
                <execution_time>0</execution_time>
            </interval>
        </default>
    </quotas>

    <users>
        <!-- 读写用户  -->
        <default>
            <password_sha256_hex>967f3bf355dddfabfca1c9f5cab39352b2ec1cd0b05f9e1e6b8f629705fe7d6e</password_sha256_hex>
            <networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
        </default>

        <!-- 只读用户  -->
        <ck>
            <password_sha256_hex>967f3bf355dddfabfca1c9f5cab39352b2ec1cd0b05f9e1e6b8f629705fe7d6e</password_sha256_hex>
            <networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>
            <profile>readonly</profile>
            <quota>default</quota>
        </ck>
    </users>
</yandex>


```



工作目录

/var/lib/clickhouse



```
clickhouse -m --host server
clickchouse -q select * from test
```



```
show databases;

create database test


CREATE TABLE test.first(
		`City` String,
    `Cost` Int16,
    `Address` IPv4,
    `CreateTIme` Datetime
)ENGINE = Memory();


//mergetree 需要主建
CREATE TABLE test.second(
		`Id` Int64,
    `Cost` Int16,
    `Address` IPv4
)ENGINE = MergeTree() order by Id;
// primary by
// order by

desc first;

show create table second;

//SETTINGS index_granularity = 8192 稀疏索引

alter table second add column age UInt8 {after column};

alter table 表名 drop column 列名

 ALTER TABLE `second` ADD COLUMN name String DEFAULT 'van';

ALTER TABLE visits MODIFY COLUMN browser Array(String)

insert into first values ('shanghai',10,'192.168.1.1');

create table newtable as oldtable;

insert into newtable  select * from oldtable

select * from first;
```





### 更新删除

```
create table tb_update(
id Int64,
name String,
age Int8
)engine=MergeTree order by id;


alter table tb_update update age = 99 where id = 1;
alter table tb_update delete where id = 1;
```



### 分区 MergeTree

https://blog.csdn.net/m0_37813354/article/details/110847747

分区是在一个表中通过指定的规则划分而成的逻辑数据集。可以按任意标准进行分区，如按月，按日或按事件类型。为了减少需要操作的数据，每个分区都是分开存储的。访问数据时，ClickHouse 尽量使用这些分区的最小子集。

```
数据分区-允许查询在指定了分区键的条件下，尽可能的少读取数据
数据分片-允许多台机器/节点同并行执行查询，实现了分布式并行计算


ClickHouse支持单机模式，也支持分布式集群模式。在分布式模式下，ClickHouse会将数据分为多个分片，并且分布到不同节点上。不同的分片策略在应对不同的SQL Pattern时，各有优势。ClickHouse提供了丰富的sharding策略，让业务可以根据实际需求选用。

1） random随机分片：写入数据会被随机分发到分布式集群中的某个节点上。

2） constant固定分片：写入数据会被分发到固定一个节点上。

3） column value分片：按照某一列的值进行hash分片。

4） 自定义表达式分片：指定任意合法表达式，根据表达式被计算后的值进行hash分片。

CREATE TABLE visits
(
    VisitDate Date,
    Hour UInt8,
    ClientID UUID
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(VisitDate)
ORDER BY Hour;

create table tb_partition(
id Int64,
name String,
age Int8,
birthday DateTime
)engine=MergeTree 
partition by toDate(birthday)
order by id;


insert into tb_partition values(1,'van',1,now() );
insert into tb_partition values(2,'fan',1,'2021-05-30 16:03:58' );


alter table {table} drop partition '1'

//复制分区 表结构需要一样
alter table {newtable} replace partition  from oldtable

optimize table {table};



分区卸载：
分区卸载后 它的物理数据没有删除，而是被转移到了当前数据目录的detached子目录 ，装载反响操作
```





视图view

```
create view user_view as select id ,name from table;

物化视图
原表删除数据 物化视图 数据不会删除
materialized
create marerialized {engine=} {populate} user_view as select id ,name from table;
```



Log引擎

```
TinyLog
没有索引 没有标记块
追加写
不允许同事读写 insert into t select * from t;


create table tb_tinylog(
id Int64,
name String
) engine= TinyLog;


Log
支持多线程
支持并发读写
mark.mrk 数据块标记

create table tb_log(
id Int64,
name String
) engine= Log;


StripeLog
数据存在一个文件
对数据建立索引
并发读写

create table tb_stripe_log(
id Int64,
name String
) engine= StripeLog;
```



### MergeTree

https://segmentfault.com/a/1190000023089140



主键是一级索引

order by 在分区内排序



（1）PARTITION BY [选填]：分区键，用于指定表数据以何种标准进行分区。分区键既可以是单个列字段，也可以通过元组的形式使用多个列字段，同时它也支持使用列表达式。如果不声明分区键，则ClickHouse会生成一个名为all的分区。合理使用数据分区，可以有效减少查询时数据文件的扫描范围。

（2）ORDER BY [必填]：排序键，用于指定在一个数据片段内，数据以何种标准排序。默认情况下主键（PRIMARY KEY）与排序键相同。排序键既可以是单个列字段，例如ORDER BY CounterID，也可以通过元组的形式使用多个列字段，例如ORDER BY（CounterID, EventDate）。当使用多个列字段排序时，以ORDERBY（CounterID, EventDate）为例，在单个数据片段内，数据首先会以CounterID排序，相同CounterID的数据再按EventDate排序。

（3）PRIMARY KEY [选填]：主键，顾名思义，声明后会依照主键字段生成一级索引，用于加速表查询。默认情况下，主键与排序键(ORDER BY)相同，所以通常直接使用ORDER BY代为指定主键，无须刻意通过PRIMARY KEY声明。所以在一般情况下，在单个数据片段内，数据与一级索引以相同的规则升序排列。与其他数据库不同，MergeTree主键允许存在重复数据（ReplacingMergeTree可以去重）。

（4）SAMPLE BY [选填]：抽样表达式，用于声明数据以何种标准进行采样。如果使用了此配置项，那么在主键的配置中也需要声明同样的表达式，例如：

order by intHash32(id)

sample by intHash32(id)

抽样表达式需要配合SAMPLE子查询使用，这项功能对于选取抽样数据十分有用。

SETTINGS: index_granularity [选填]:index_granularity对于MergeTree而言是一项非常重要的参数，它表示索引的粒度，默认值为8192。也就是说，MergeTree的索引在默认情况下，每间隔8192行数据才生成一条索引，其具体声明方式如下所示：

Setting index_granularity = 8192





```
按主建排序
分布式
order by 主建必须在前面
主键没有唯一要求，不能去重

create table tb_merge_tree(
	uid Int64,
	name String,
	birthday Date
)engine= MergeTree
partition by birthday
primary key uid
order by (uid,birthday);


```



二级索引

```
CREATE TABLE skip_test (
    ID String,
    URL String,
    Code String,
    EventTime Date,
    INDEX a ID TYPE minmax GRANULARITY 5,
    INDEX b (length(ID) * 8) TYPE set(2) GRANULARITY 5
    INDEX c (ID,Code) TYPE ngrambf_v1(3,256,2,0) GRANULARITY 5,
    INDEX d ID TYPE tokenbf_v1(256,2,0) GRANULARITY 5
) ENGINE = MergeTree()
ORDER BY ID 

（1）minmax:minmax索引记录了一段数据内的最小和最大极值，其索引的作用类似分区目录的minmax索引，能够快速跳过无用的数据区间，示例如下所示：
INDEX a ID TYPE minmax GRANULARITY 5
上述示例中minmax索引会记录这段数据区间内ID字段的极值。极值的计算涉及每5个index_granularity区间中的数据。

（2）set:set索引直接记录了声明字段或表达式的取值（唯一值，无重复），其完整形式为set(max_rows)，其中max_rows是一个阈值，表示在一个index_granularity内，索引最多记录的数据行数。如果max_rows=0，则表示无限制，例如：
INDEX b (length(ID) * 8) TYPE set(2) GRANULARITY 5
上述示例中set索引会记录数据中ID的长度 * 8后的取值。其中，每个index_granularity内最多记录100条。

（3）ngrambf_v1:ngrambf_v1索引记录的是数据短语的布隆表过滤器，只支持String和FixedString数据类型。ngrambf_v1只能够提升in、notIn、like、equals和notEquals查询的性能，其完整形式为ngrambf_v1(n,size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)。这些参数是一个布隆过滤器的标准输入，如果你接触过布隆过滤器，应该会对此十分熟悉。它们具体的含义如下：
❑ n:token长度，依据n的长度将数据切割为token短语。
❑ size_of_bloom_filter_in_bytes：布隆过滤器的大小。
❑ number_of_hash_functions：布隆过滤器中使用Hash函数的个数。❑ random_seed: Hash函数的随机种子。

例如在下面的例子中，ngrambf_v1索引会依照3的粒度将数据切割成短语token，token会经过2个Hash函数映射后再被写入，布隆过滤器大小为256字节。
INDEX c (ID,Code) TYPE ngrambf_v1(3,256,2,0) GRANULARITY 5,
（4）tokenbf_v1:tokenbf_v1索引是ngrambf_v1的变种，同样也是一种布隆过滤器索引。tokenbf_v1除了短语token的处理方法外，其他与ngrambf_v1是完全一样的。tokenbf_v1会自动按照非字符的、数字的字符串分割token，具体用法如下所示：
INDEX d ID TYPE tokenbf_v1(256,2,0) GRANULARITY 5
```

ReplacingMergeTree

```
删除具有相同（区内）重复的主键数据 不能保证没有重复数据 只有在合并的时候 才会去重
使用order by排序健作为判断重复数据的唯一依据

如果没有设置版本号 保留同组中重复数组中的最后一行
如果设置了版本号 保留版本最大的一行

create table tb_replace_merge_tree(
	uid Int64,
	name String,
	birthday Date
)engine= ReplacingMergeTree 
primary key uid
order by uid;

create table tb_replace_merge_tree(
	uid Int64,
	name String,
	birthday Date,
	version Uint8
)engine= ReplacingMergeTree (version)
primary key uid
order by uid;

create table tb_replace_merge_tree(
	uid Int64,
	name String,
	birthday Date,
	ctime DateTime
)engine= ReplacingMergeTree (ctime)
primary key uid
order by uid;

optimize table {name} final;
```



CollapsingMergeTree

```
以增代删 通过sign标记要删除数据


如果sign -1表示删除
create table tb_collapsing_mrege_tree(
	uid Int64,
	name String,
	birthday Date,
	sign Int8
)engine= CollapsingMergeTree (sign)
primary key uid
order by uid;
```





### AggregatingMergeTree

https://blog.csdn.net/u012551524/article/details/112300519

```
ClickHouse 会将一个数据片段内所有具有相同主键（准确的说是 排序键）的行替换成一行，这一行会存储一系列聚合函数的状态。
可以使用 AggregatingMergeTree 表来做增量数据的聚合统计，包括物化视图的数据聚合。



CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = AggregatingMergeTree()
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[TTL expr]
[SETTINGS name=value, ...]



create table test_agg (
	shop_code String,
	product_code String,
	name AggregateFunction(uniq,String),
	out_count AggregateFunction(sum,Int),
	write_date DateTime
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(write_date)
ORDER BY (shop_code,product_code)
PRIMARY KEY shop_code;


insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-12-17 21:18:00';
insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-12-17 21:18:00';
insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-11-17 21:18:00';
insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-12-17 21:18:00';
insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-11-17 21:18:00';
insert into test_agg select '1','短袖',uniqState('monica'),sumState(toInt32(100)),'2020-12-17 21:18:00';

select shop_code,product_code,uniqMerge(name),sumMerge(out_count) from test_agg group by shop_code,product_code;


create table test_table (
	shop_code String,
	product_code String,
	name String,
	out_count Int,
	write_date DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(write_date)
ORDER BY (shop_code,product_code);

create materialized view test_agg_view 
ENGINE = AggregatingMergeTree()
PARTITION BY shop_code
ORDER BY (shop_code,product_code)
AS SELECT
shop_code,
product_code,
uniqState(name) as name,
sumState(out_count) as out_count
FROM test_table
GROUP BY shop_code,product_code;


insert into test_table values 
('1','短袖','tracy',100,'2020-12-17 21:18:00'),
('1','短袖','tracy',100,'2020-12-18 21:18:00'),
('1','外套','tracy',100,'2020-12-19 21:18:00');

select shop_code,uniqMerge(name),sumMerge(out_count) from test_agg_view group by shop_code,product_code;


使用ORDER BY排序键作为聚合数据的依据
使用AggregateFunction字段类型定义聚合函数的类型以及聚合字段
只有在合并分区的时候才会触发聚合计算的逻辑
聚合只会发生在同分区内，不同分区的数据不会发生聚合
在进行数据计算时，因为同分区的数据已经基于ORDER BY排序，所以能够找到相邻且具有相同聚合key的数据
在聚合数据时，同一分区内，相同聚合key的多行数据会合并成一行，对于那些非主键、非AggregateFunction类型字段，则会取第一行数据
AggregateFunction类型字段使用二进制存储，在写入数据时，需要调用state函数；在读数据时，需要调用merge函数，*表示定义时使用的聚合函数
AggregateMergeTree通常作为物化视图的引擎，与普通的MergeTree搭配使用

```







Optimize 

```
OPTIMIZE TABLE [db.]name [ON CLUSTER cluster] [PARTITION partition | PARTITION ID 'partition_id'] [FINAL] [DEDUPLICATE [BY expression]]

If OPTIMIZE does not perform a merge for any reason, it does not notify the client. To enable notifications, use the optimize_throw_if_noop setting.
If you specify a PARTITION, only the specified partition is optimized. How to set partition expression.
If you specify FINAL, optimization is performed even when all the data is already in one part. Also merge is forced even if concurrent merges are performed.
If you specify DEDUPLICATE, then completely identical rows (unless by-clause is specified) will be deduplicated (all columns are compared), it makes sense only for the MergeTree engine.

```







集群

https://zhuanlan.zhihu.com/p/161242274



https://github.com/fyqtian/clickhouse-cluster



https://www.jianshu.com/p/d1842290bd48

```
CREATE DATABASE company_db ON CLUSTER 'company_cluster';

CREATE TABLE company_db.events ON CLUSTER 'company_cluster' (
    time DateTime,
    uid  Int64,
    type LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/table', '{replica}')
PARTITION BY toDate(time)
ORDER BY (uid);

CREATE TABLE company_db.events_distr ON CLUSTER 'company_cluster' AS company_db.events
ENGINE = Distributed('company_cluster', company_db, events, uid);
                                       



create database tutorial cluster on company_db;

CREATE TABLE tutorial.hits_local(
id Int8,
name String
)ENGINE = MergeTree() 
order by id;


CREATE TABLE tutorial.hits_all AS tutorial.hits_local
ENGINE = Distributed(company_cluster, tutorial, hits_local, rand());
                                      库         表
insert into hits_local values (1,'fan');
```



### 分片副本

https://segmentfault.com/a/1190000023098408

不包含副本的分片
如果直接使用**node标签定义分片节点**，那么该集群将只包含分片，不包含副本。以下面的配置为例



查看配置文件宏

select * from system.macros; 



远程节点

select * from remote('clickhouse02:9000','system','macros','default');

```
create database test  on CLUSTER  company_cluster;

create table test.tb on CLUSTER  company_cluster(
	uid Int64,
	name String,
	birthday Date
)engine= ReplicatedMergeTree('/clickhouse/tables/{shard}/table_name', '{replica'}) 
primary key uid
order by uid;

ENGINE =ReplicatedMergeTree('zk_path','replica_name')
对于zk_path而言，同一张数据表的同一个分片的不同副本，应该定义相同的路径；
而对于replica_name而言，同一张数据表的同一个分片的不同副本，应该定义不同的名称。
变量维护在配置文件中



建库 和表 必须用 on cluster 
CREATE TABLE test.tb1 on cluster company_cluster 
(
    EventDate DateTime,
    CounterID UInt32,
    UserID UInt32
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/1-{shard}/tb1', '{replica}')
PARTITION BY toYYYYMM(EventDate)
ORDER BY (CounterID, EventDate, intHash32(UserID))
SAMPLE BY intHash32(UserID);


CREATE TABLE test.hits_all on cluster company_cluster  AS test.tb1
ENGINE = Distributed(company_cluster, test, tb1, rand());





CREATE TABLE test.tb2 on cluster company_cluster 
(
    Name String,
    CounterID UInt32,
    UserID UInt32
) ENGINE = MergeTree()
PARTITION BY Name
ORDER BY (CounterID, Name, intHash32(UserID));

CREATE TABLE test.hits_tb2 on cluster company_cluster  AS test.tb2

ENGINE = Distributed(company_cluster, test, tb2, rand());

❑ cluster：集群名称，与集群配置中的自定义名称相对应。在对分布式表执行写入和查询的过程中，它会使用集群的配置信息来找到相应的host节点。
❑ database和table：分别对应数据库和表的名称，分布式表使用这组配置映射到本地表。
❑ sharding_key：分片键，选填参数。在数据写入的过程中，分布式表会依据分片键的规则，将数据分布到各个host节点的本地表。

insert into test.tb2 values('no1',1,1);

```



```
create table test.t1 on CLUSTER  test(
	uid Int64,
	name String,
	birthday Date,
	creatdAt DateTime
)engine= MergeTree
partition by birthday
ttl creatdAt + INTERVAL 5 MINUTE
order by birthday;
insert into test.t1 values (13, 'n', '2001-01-01', now());
```





```
create database on cluster test;


create table test.t3 on CLUSTER  test(
	city String,
	isp String,
	cost Int16,
	lost Int8,
	creatdAt DateTime
)engine= MergeTree
partition by city
ttl creatdAt + INTERVAL 360 MINUTE
order by creatdAt;

INSERT INTO test.t3
  SELECT 'shanghai','dianxin',rand() % 100000000,rand() % 100000000,now() FROM system.numbers
  LIMIT 10;


INSERT INTO test.t3
  SELECT 'shanghai','yidong',rand() % 100000000,rand() % 100000000,now() FROM system.numbers
  LIMIT 10;
  

select sum(lost) ,isp  from t3 group by isp
select sum(lost) /count(),isp ,count() as c from t3 group by isp
  
create  MATERIALIZED view test.t3_view on CLUSTER  test
ENGINE = AggregatingMergeTree
POPULATE
as select from test.t3
```



https://blog.csdn.net/vkingnew/article/details/106775064

https://altinity.com/blog/2018/10/16/updates-in-clickhouse

```
CREATE TABLE download (
  when DateTime,
  userid UInt32,
  bytes Float32
) ENGINE=MergeTree
PARTITION BY toYYYYMM(when)
ORDER BY (userid, when);

INSERT INTO download
  SELECT
    now() + number * 60 as when,
    25,
    rand() % 100000000
  FROM system.numbers
  LIMIT 5000;


计算：每个用户每天下载的次数和流量：
SELECT
  toStartOfDay(when) AS day,
  userid,
  count() as downloads,
  sum(bytes) AS bytes
FROM download
GROUP BY userid, day
ORDER BY userid, day;

CREATE MATERIALIZED VIEW download_daily_mv
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(day) ORDER BY (userid, day)
POPULATE
AS SELECT
  toStartOfDay(when) AS day,
  userid,
  count() as downloads,
  sum(bytes) AS bytes
FROM download
GROUP BY userid, day


```

# SummingMergeTree

https://gaokaiyang.blog.csdn.net/article/details/112118734

```
create table test_summing (
shop_code String,
product_code String,
in_count Int,
out_count Int,
day Int16,
write_date DateTime
) ENGINE = SummingMergeTree()
ORDER BY (shop_code,product_code,day)
PRIMARY KEY shop_code;


insert into test_summing values ('1','短袖',100,100,2000,'2020-12-17 21:18:00');

insert into test_summing values ('1','短袖',100,200,2000,'2020-12-18 21:19:00');

insert into test_summing values ('1','短袖',100,100,2000,'2020-12-17 21:20:00');

insert into test_summing values ('2','外套',100,100,1000,'2020-12-17 21:18:00');

insert into test_summing values ('2','外套',100,100,1000,'2020-12-17 21:19:00');

```



