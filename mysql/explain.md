### explain用法

https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

https://segmentfault.com/a/1190000021458117?utm_source=tag-newest



### explain的用途

```
1. 表的读取顺序如何
2. 数据读取操作有哪些操作类型
3. 哪些索引可以使用
4. 哪些索引被实际使用
5. 表之间是如何引用
```



### explain的执行效果

```
mysql> explain select * from subject where id = 1 \G
******************************************************
           id: 1
  select_type: SIMPLE
        table: subject
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
******************************************************
```



## explain包含的字段

```
1. id 								//select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序
2. select_type 						//查询类型
3. table 							//正在访问哪个表
4. partitions 						//匹配的分区
5. type 							//访问的类型
6. possible_keys					//显示可能应用在这张表中的索引，一个或多个，但不一定实际使用到
7. key 								//实际使用到的索引，如果为NULL，则没有使用索引
8. key_len 							//表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度
9. ref 								//显示索引的哪一列被使用了，如果可能的话，是一个常数，哪些列或常量被用于查找索引列上的值
10. rows							//根据表统计信息及索引选用情况，大致估算出找到所需的记录所需读取的行数
11. filtered						//查询的表行占表的百分比
12. Extra 							//包含不适合在其它列中显示但十分重要的额外信息
```

![2864885534-202c0878c1abf896_articlex](D:\gocode\read-code\images\2864885534-202c0878c1abf896.png)

#### type类型

```
NULL>system>const>eq_ref>ref>ref_or_null>index_merge>range>index>ALL

NULL
MySQL能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引
例子：
explain select min(id) from subject;

system
表只有一行记录（等于系统表），这是const类型的特列，平时不大会出现，可以忽略

const
表示通过索引一次就找到了，const用于比较primary key或uique索引，因为只匹配一行数据，所以很快，如主键置于where列表中，MySQL就能将该查询转换为一个常量
例子：
explain select * from teacher where teacher_no = 'T2010001';

eq_ref
唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描 非null
例子：
explain select subject.* from subject left join teacher on subject.teacher_id = teacher.id;

ref
非唯一性索引扫描，返回匹配某个单独值的所有行
本质上也是一种索引访问，返回所有匹配某个单独值的行
然而可能会找到多个符合条件的行，应该属于查找和扫描的混合体
例子：
explain select subject.* from subject,student_score,teacher where subject.id = student_id and subject.teacher_id = teacher.id;

ref_or_null
类似ref，但是可以搜索值为NULL的行
例子：
explain select * from teacher where name = 'wangsi' or name is null;

index_merge
表示使用了索引合并的优化方法
例子：
explain select * from teacher where id = 1 or teacher_no = 'T2010001' .

range
只检索给定范围的行，使用一个索引来选择行，key列显示使用了哪个索引
一般就是在你的where语句中出现between、<>、in等的查询。
例子：
explain select * from subject where id between 1 and 3;


index
Full index Scan，Index与All区别：index只遍历索引树，通常比All快
因为索引文件通常比数据文件小，也就是虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘读的。
例子：
explain select id from subject;

ALL
Full Table Scan，将遍历全表以找到匹配行
例子：
explain select * from subject;


possible_keys字段
显示可能应用在这张表中的索引，一个或多个
查询涉及到的字段若存在索引，则该索引将被列出，但不一定被实际使用

key字段
实际使用到的索引，如果为NULL，则没有使用索引
查询中若使用了覆盖索引（查询的列刚好是索引），则该索引仅出现在key列表

key_len字段
表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度
在不损失精确度的情况下，长度越短越好
key_len显示的值为索引字段最大的可能长度，并非实际使用长度
即key_len是根据定义计算而得，不是通过表内检索出的

ref字段
显示索引的哪一列被使用了，如果可能的话，是一个常数，哪些列或常量被用于查找索引列上的值


```

### Extra字段

包含不适合在其它列中显示但十分重要的额外信息

 	Using filesort
 	说明MySQL会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取
 	MySQL中无法利用索引完成的排序操作称为“文件排序”
 	例子：
 	explain select * from subject order by name;
 	
 	Using temporary
 	使用了临时表保存中间结果，MySQL在对结果排序时使用临时表，常见于排序order by 和分组查询group by
 	例子：
 	explain select subject.* from subject left join teacher on subject.teacher_id = teacher.id
 	 -> union 
 	 -> select subject.* from subject right join teacher on subject.teacher_id = teacher.id


 	 

 	Using index
 	表示相应的select操作中使用了覆盖索引（Covering Index）,避免访问了表的数据行，效率不错！
 	如果同时出现using where，表明索引被用来执行索引键值的查找
 	如果没有同时出现using where，表明索引用来读取数据而非执行查找动作
 	例子：
 	explain select subject.* from subject,student_score,teacher where subject.id = student_id and subject.teacher_id = teacher.id;
 	备注：
 	覆盖索引：select的数据列只用从索引中就能够取得，不必读取数据行，MySQL可以利用索引返回select列表中的字段，而不必根据索引再次读取数据文件，即查询列要被所建的索引覆盖
 	
 	Using where
 	使用了where条件
 	例子：
 	explain select subject.* from subject,student_score,teacher where subject.id = student_id and s


### 慢查询优化基本步骤

0.先运行看看是否真的很慢，注意设置SQL_NO_CACHE

1.where条件单表查，**锁定最小返回记录表**。这句话的意思是把查询语句的where都应用到表中返回的记录数最小的表开始查起，单表每个字段分别查询，看哪个字段的区分度最高

2.explain查看执行计划，是否与1预期一致（从锁定记录较少的表开始查询）

3**.order by limit 形式的sql语句让排序的表优先查**

6.观察结果，不符合预期继续从0分析