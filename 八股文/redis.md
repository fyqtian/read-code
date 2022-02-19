https://mp.weixin.qq.com/s/wrrXz4GoILd5hsbrYACTmA

https://github.com/rbmonster/learning-note/blob/master/src/main/java/com/toc/REDIS.md



### zipList 压缩列表

压缩列表是 List 、hash、 sorted Set 三种数据类型底层实现之一。

ziplist 是由一系列特殊编码的连续内存块组成的顺序型的数据结构，ziplist 中可以包含多个 entry 节点，每个节点可以存放整数或者字符串。

ziplist 在表头有三个字段 zlbytes、zltail 和 zllen，分别表示列表占用字节数、列表尾的偏移量和列表中的 entry 个数；压缩列表在表尾还有一个 zlend，表示列表结束。



如果我们要查找定位**第一个元素和最后一个元素**，可以通过表头三个字段的长度直接定位，复杂度是 O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是 O(N)





### skipList 跳跃表

sorted set 类型的排序功能便是通过「跳跃列表」数据结构来实现。

跳跃表（skiplist）**是一种有序数据结构**，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

跳跃表支持平均 O（logN）、最坏 O（N）复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

跳表在链表的基础上，增加了多层级索引，通过索引位置的几个跳转，实现数据的快速定位，如下图所示：



### 整数数组（intset）

当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis 就会使用整数集合作为集合键的底层实现。结构如下：



**String**：存储数字的话，采用 int 类型的编码，如果是非数字的话，采用 raw 编码；

**Hash**：Hash 对象的编码可以是 ziplist 或 hashtable。

当 Hash 对象同时满足以下两个条件时，Hash 对象采用 ziplist 编码：

- **Hash 对象保存的所有键值对的键和值的字符串长度均小于 64 字节。**
- **Hash 对象保存的键值对数量小于 512 个。**



**List**：List 对象的编码可以是 ziplist 或 linkedlist，字符串长度 < 64 字节且元素个数 < 512 使用 ziplist 编码，否则转化为 linkedlist 编码；

**Redis中的列表list，在版本3.2之前，列表底层的编码是ziplist和linkedlist实现的，但是在版本3.2之后，重新引入 quicklist，列表的底层都由quicklist实现。**

quickList是一个ziplist组成的linkedlist双向链表。每个节点使用ziplist来保存数据，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来。

- ziplist 是一个特殊的双向链表,特殊之处在于没有维护双向指针:prev next；而是存储上一个 entry的长度和 当前entry的长度，通过长度推算下一个元素在什么地方。

- ziplist使用连续的内存块。

- linkedList 便于在表的两端进行push和pop操作，在插入节点上复杂度很低，但是它的内存开销比较大。

  - 它在每个节点上除了要保存数据之外，还要额外保存两个指针；
  - 其次，双向链表的各个节点是单独的内存块，地址不连续，节点多了容易产生内存碎片。

  

1. 列表对象保存的所有字符串元素的长度都小于64字节。
2. 列表对象保存的元素数量小于512个

```
list-max-ziplist-entries 512
list-max-ziplist-value 64
```



**Set**：Set 对象的编码可以是 intset 或 hashtable，intset 编码的对象使用整数集合作为底层实现，把所有元素都保存在一个整数集合里面。

保存元素为整数且元素个数小于一定范围使用 intset 编码，任意条件不满足，则使用 hashtable 编码；



**Zset**：Zset 对象的编码可以是 ziplist 或 zkiplist，当采用 ziplist 编码存储时，每个集合元素使用两个紧挨在一起的压缩列表来存储。

```
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
```





### 内存淘汰机制[[Top\]](https://github.com/rbmonster/learning-note/blob/master/src/main/java/com/toc/REDIS.md#index)

内存淘汰机制：八种大体上可以分为4中，lru（最近最少使用）、lfu（最少使用频率）、random（随机）、ttl（根据生存时间，快过期）。

1. volatile-lru：从已设置过期时间的数据集中挑选最近最少使用的数据淘汰。
2. volatile-ttl：从已设置过期时间的数据集中挑选将要过期的数据淘汰。
3. volatile-random：从已设置过期时间的数据集中任意选择数据淘汰。
4. volatile-lfu：从已设置过期时间的数据集挑选使用频率最低的数据淘汰。
5. allkeys-lru：从数据集中挑选最近最少使用的数据淘汰
6. allkeys-lfu：从数据集中挑选使用频率最低的数据淘汰。
7. allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
8. no-enviction（驱逐）：禁止驱逐数据，这也是默认策略。意思是当内存不足以容纳新入数据时，新写入操作就会报错，请求可以继续进行，线上任务也不能持续进行，采用no-enviction策略可以保证数据不被丢失。



### LRU实现[[Top\]](https://github.com/rbmonster/learning-note/blob/master/src/main/java/com/toc/REDIS.md#index)

常规的LRU算法会维护一个双向链表，用来表示访问关系，且需要额外的存储存放 next 和 prev 指针，牺牲比较大的存储空间。

Redis的实现LRU会维护一个全局的LRU时钟，并且每个键中也有一个时钟，每次访问键的时候更新时钟值。

淘汰过程：Redis会基于server.maxmemory_samples配置选取固定数目的key，然后比较它们的lru访问时间，然后淘汰最近最久没有访问的key，maxmemory_samples的值越大，Redis的近似LRU算法就越接近于严格LRU算法，但是相应消耗也变高，对性能有一定影响，样本值默认为5。



```go
type data struct {
	key, val int
}
type LRUCache struct {
	cache map[int]*list.Element
	*list.List
	capacity int
}

func Constructor(capacity int) LRUCache {
	return LRUCache{
		cache:    map[int]*list.Element{},
		List:     list.New(),
		capacity: capacity,
	}
}

func (this *LRUCache) Get(key int) int {
	val, ok := this.cache[key]
	if !ok {
		return -1
	}
	this.List.MoveToFront(val)
	return val.Value.(data).val
}

func (this *LRUCache) Put(key int, value int) {
	e, ok := this.cache[key]
	if !ok {
		e = this.List.PushFront(data{key, value})
		this.cache[key] = e
	}else {
		e.Value = data{key, value}
	}
	this.List.MoveToFront(e)
	if len(this.cache) > this.capacity {
		rs := this.List.Back()
		this.List.Remove(rs)
		delete(this.cache, rs.Value.(data).key)
	}
}

```





### 集群下与客户端交互[[Top\]](https://github.com/rbmonster/learning-note/blob/master/src/main/java/com/toc/REDIS.md#index)

#### MOVED错误[[Top\]](https://github.com/rbmonster/learning-note/blob/master/src/main/java/com/toc/REDIS.md#index)

键命令执行步骤主要分两步：

1. **计算槽。Redis首先需要计算键所对应的槽。根据键的有效部分使用CRC16函数计算出散列值，再取对16383的余数，使每个键都可以映射到0~16383槽范围内。如指令`127.0.0.1:6379> cluster keyslot key:test:111`**
2. 槽节点查找。Redis计算得到键对应的槽后，需要查找槽所对应的节点。集群内通过消息交换每个节点都会知道所有节点的槽信息，**内部保存在clusterState结构中。**
3. **若节点的槽不是当前节点，返回MOVED重定向错误。**



使用redis-cli命令时，**可以加入-c参数支持自动重定向，简化手动发起重定向操作**，如下所示：
redis-cli自动帮我们连接到正确的节点执行命令，这个过程是在redis-cli内部维护，实质上是client端接到MOVED信息之后再次发起请 求，并不在Redis节点中完成请求转发，如下图所示





#### ASK 错误[[Top\]](https://github.com/rbmonster/learning-note/blob/master/src/main/java/com/toc/REDIS.md#index)

ASK重定向：在线迁移槽（slot）的过程中，客户端向slot发送请求，若键对象不存在，则可能存在于目标节点，这时源节点会回复 ASK重定向异常。
格式如下：(error) ASK {slot} {targetIP}:{targetPort}

> 客户端从ASK重定向异常提取出目标节点信息，发送asking命令到目标节点打开客户端连接标识，再执行键命令。如果存在则执行，不存在则返 回不存在信息



1. 节点数据库和单机数据库在数据库方面的一个区别是，**节点只能使用0号数据库**，而单机Redis服务器则没有这个限制。



Redis 集群并没有直接使用一致性哈希，**而是使用了哈希槽 （slot） 的概念。没有使用Hash算法，而是使用了crc16校验算法。**槽位其实就是一个个的空间的单位。

> 每个key经过crc16校验算法计算，会落在对应的哈希槽上，便可以定位到节点的redis

###  