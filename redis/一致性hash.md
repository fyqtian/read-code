一致性hash

https://blog.csdn.net/qq_21125183/article/details/90019034



https://www.jianshu.com/p/6ad87a1f070e

https://blog.csdn.net/tr1912/article/details/81265007 集群故障转移

一致性哈希的空间是一个圆环（2的32次方-1），节点分布是基于圆环的，无法很好的控制数据分布（数据分布不规则）

而 redis cluster 的槽位空间是自定义分配的hash算法**cr16**



CRC16(key) & 16383
当 key 包含 hash tags 的时候**（例如 key{sub}1）**，会以 sub tags **中指定的字符串（就是 sub ）计算槽**，所以key{sub}1和key{sub}2会到同一个槽中。

如果当前的key所在的槽不在当前节点，会发送moved指令 指向槽所在的节点





redis cluster 包含了**16384**个哈希槽，每个 key 通过计算后都会落在具体一个槽位上，而这个**槽位是属于哪个存储节点**的，则由用户自己定义分配。例如机器硬盘小的，可以分配少一点槽位，硬盘大的可以分配多一点。如果节点硬盘都差不多则可以平均分配。所以哈希槽这种概念很好地解决了一致性哈希的弊端。



另外在容错性和扩展性上，表象与一致性哈希一样，**都是对受影响的数据进行转移**。而哈希槽本质上是对槽位的转移，把故障节点负责的槽位转移到其他正常的节点上。扩展节点也是一样，把其他节点上的槽位转移到新的节点上。



但一定要注意的是，对于槽位的转移和分派，**redis 集群是不会自动进行的**，**而是需要人工配置的**。所以 redis 集群的高可用是依赖于节点的**主从复制与主从间的自动故障转移**。



