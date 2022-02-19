## DNS

nslookup 域名 dns服务器地址

nslookup -qt=type domain [dns-server]

nslookup baidu.com 127.0.0.1 -port=9898



dig @8.8.8.8（dns服务器）  [www.baidu.com](http://www.baidu.com/) A(记录类型)



dig +trace www.scnu.edu.com

查询ns



https://zhuanlan.zhihu.com/p/88260838

http://c.biancheng.net/view/6457.html

https://www.cnblogs.com/joycamp/p/13498104.html

1、 **什么是DNS协议？**

1. DNS协议就是用来将**域名解析到IP**地址的一种协议，当然，也可以将**IP地址转换为域名**的一种协议。
2. DNS协议基**于UDP和TCP**协议的，端口号**53**，用户到服务器采用UDP，DNS**服务器通信采用TCP**
3. 大型运营商、互联网机构等会向公众提供免费的DNS服务，例如，谷歌的8.8.8.8 8.8.4.4 阿里巴巴223.5.5.5 223.6.6.6
4. 如果DNS服务器down掉了，那么你只能通过IP地址来访问服务了。
5. 我们从以下几部分来理解DNS协议：







## 域名结构

**像Linux目录结构一样**，现代因特网采用层次**树状结构**的命名方法，任何一个连接在因特网上的主机或路由器，都有一个唯一的层次结构的名字，该名字称为域名。

例如：xxx.yyy.zzz.com
从右边的**com是顶级域名**，到左依次是：二级域名，三级域名，四级域名

域名的分级：域名可以**划分为各个子域**，子域还可以继续划分为**子域的子域**，这样就形成了顶级域、二级域、三级域等。

其中顶级域名分为：国家顶级域名、通用顶级域名、反向域名。

```
国家顶级域名：中国:cn， 美国:us，英国uk...
通用顶级域名：com 公司企业  edu教育机构 gov政府部门  int国际组织  mil军事部门  net网络 org非盈利组织...
反向域名：只有一个arpa，用于PTR查询（IP地址转换为域名） 。 
```

### 域名服务器

域名服务器主要分为：**根域名**服务器、**顶级域名**服务器、**权限域名**服务器、**本地域名**服务器。

先举个例子来看一下各个服务器之间的联系：

当一个应用要通过DNS来查询某个主机名，比如www.google.com的ip时，粗略地说，查询过程是这样的：**它先与根服务器之一联系**，根服务器根据顶级域名com，会响应命名空间为com的顶级域服务器的ip；于是该应用接着向com顶级域服务器发出请求，com顶级域服务器会响应命名空间为google.com的权威DNS服务器的ip地址；最后该应用将请求命名空间为google.com的权威DNS服务器，该权威DNS服务器会响应主机名为www.google.com的ip。

实际上，除了上图层次结构中所展示的DNS外，还有一类与我们接触更为密切的DNS服务器，**它们是本地DNS服务器**，我们经常在电脑上配置的DNS服务器通常就是此类。它们一般由某公司，某大学，或某居民区提供，比如Google提供的DNS服务器8.8.8.8；比如常被人诟病的114.114.114.114等。



**根域名服务器**
根服务器主要用来管理互联网的主目录。
所有根服务器均由美国政府授权的互联网域名与号码分配机构ICANN统一管理，负责全球互联网域名根服务器、域名体系和IP地址等的管理。
全球共有**13台根**服务器（实际有镜像）。1个为根服务器架构主根服务器，放置在美国。其余12个均为辅根服务器，其中9个放置在美国，欧洲2个，位于英国和瑞典，亚洲1个，位于日本。 据说，在主根服务器系统上还有一个更高级的、隐藏着的母服务器，当然也在美国，而全世界所有的顶级域名都是由这台母服务器来确定的。
中国还没有自己的根服务器。都是根服务器的镜像（5个）



### 域名查询

查询方式分为**递归查询**和**迭代查询**



当客户机提出查询请求时，**首先在本地计算机的缓存中查找**，如果在本地无法查询信息，则将查询请求发给DNS服务器

2）首先客户机将域名查询请求发送到**本地DNS服务器**（局域网），当本地DNS服务器接到查询后，首先在该服务器管理的区域的记录中查找，如果找到该记录，则进行此记录进行解析，如果没有区域信息可以满足查询要求，服务器在本地缓存中查找

3） 如果本地服务器不能在本地找到客户机查询的信息，将客户机请求发送到**根域名DNS服务器**

4） 根域名服务器负责解析客户机请求的根域名部分，它将包**含下一级域名信息的DNS服务器地址地址**，返回给客户机的DNS服务器地址

5） 客户机的DNS服务器利用根域名服务器解析的地址访问下一级DNS服务器，得到再下一级域名的DNS服务器地址

6） 按照上述递归方法逐级接近查询目标，最后在有目标域名的DNS服务器上找到相应IP地址信息

7） 客户机的本地DNS服务器将递归查询结构返回客户机

8） 客户机利用从本地DNS服务器查询得到的IP访问目标主机，就完成了一个解析过程

9） 同时**客户机本地DNS服务器更新其缓存表**，客户机也更新期缓存表，方便以后查询





### DNS实现

https://www.cnblogs.com/dongkuo/p/6714071.html

[DNS协议详解及报文格式分析_xianjian1990的博客-CSDN博客](https://blog.csdn.net/xianjian1990/article/details/104182896)



DNS服务器存储的**资源记录（Resource Records，RRs）**，一条资源记录(RR)记载着一个映射关系。每条RR通常包含如下表所示的一些信息：

|   字段   |       含义        |
| :------: | :---------------: |
|   NAME   |       名字        |
|   TYPE   |       类型        |
|  CLASS   |        类         |
|   TTL    |     生存时间      |
| RDLENGTH | RDATA所占的字节数 |
|  RDATA   |       数据        |

NAME和RDATA表示的含义根据TYPE的取值不同而不同，常见的：

1. 若TYPE=A，则name是主机名，value是其对应的ip；
2. 若TYPE=NS，则name是一个域，value是一个权威DNS服务器的主机名。该记录表示name域的域名解析将由value主机名对应的DNS服务器来做；
3. 若TYPE=CNAME，则value是别名为name的主机对应的规范主机名；
4. 若TYPE=MX，则value是别名为name的邮件服务器的规范主机名；
5. ……
6. 

**TYPE**实际上还有其他类型，所有可能的type及其约定的数值表示如下：

| TYPE  | value |                       meaning                       |
| :---: | :---: | :-------------------------------------------------: |
|   A   |   1   |                   a host address                    |
|  NS   |   2   |            an authoritative name server             |
|  MD   |   3   |       a mail destination (Obsolete - use MX)        |
|  MF   |   4   |        a mail forwarder (Obsolete - use MX)         |
| CNAME |   5   |           the canonical name for an alias           |
|  SOA  |   6   | marks the start of a zone of authority 说明ns主记录 |
|  MB   |   7   |        a mailbox domain name (EXPERIMENTAL)         |
|  MG   |   8   |         a mail group member (EXPERIMENTAL)          |
|  MR   |   9   |      a mail rename domain name (EXPERIMENTAL)       |
| NULL  |  10   |              a null RR (EXPERIMENTAL)               |
|  WKS  |  11   |          a well known service description           |
|  PTR  |  12   |                a domain name pointer                |
| HINFO |  13   |                  host information                   |
| MINFO |  14   |          mailbox or mail list information           |
|  MX   |  15   |                    mail exchange                    |
|  TXT  |  16   |                    text strings                     |



### 查询类：通常为1，表明是Internet数据  class





DNS**请求与响应的格式是一致的**，其整体分为Header、Question、Answer、Authority、Additional5部分，如下图所示：

![](https://blog-1251252991.cos.ap-chengdu.myqcloud.com/2017-04-15%2003-02-39%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

Header部分是一定有的，长度固定为**12个字节**；其余4部分可能有也可能没有，并且长度也不一定，这个在Header部分中有指明。Header的结构如下：

![](https://blog-1251252991.cos.ap-chengdu.myqcloud.com/2017-04-15%2003-06-31%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

1. ID：**占16位**。该值由发出DNS请求的程序生成，DNS服务器在响应时会使用该ID，这样便于请求程序区分不同的DNS响应。
2. QR：占1位。指示该消息是请求还是响应。0表示请求；1表示响应。
3. OPCODE：占4位。指示请求的类型，有请求发起者设定，响应消息中复用该值。0表示标准查询；1表示反转查询；2表示服务器状态查询。3~15目前保留，以备将来使用。
4. AA（Authoritative Answer，权威应答）：占1位。表示响应的服务器是否是权威DNS服务器。只在响应消息中有效。
5. TC（TrunCation，截断）：占1位。指示消息是否因为传输大小限制而被截断
6. RD（Recursion Desired，期望递归）：占1位。该值在请求消息中被设置，响应消息复用该值。如果被设置，**表示希望服务器递归查询**。但服务器不一定支持递归查询。
7. RA（Recursion Available，递归可用性）：占1位。**该值在响应消息中被设置或被清除，以表明服务器是否支持递归查询**。
8. Z：占3位。保留备用。
9. RCODE（Response code）：占4位。该值在响应消息中被设置。取值及含义如下：
   - 0：No error condition，没有错误条件；
   - 1：Format error，请求格式有误，服务器无法解析请求；
   - 2：Server failure，服务器出错。
   - 3：Name Error，只在权威DNS服务器的响应中有意义，表示请求中的域名不存在。
   - 4：Not Implemented，服务器不支持该请求类型。
   - 5：Refused，服务器拒绝执行请求操作。
   - 6~15：保留备用。
10. QDCOUNT：占16位（无符号）。指明Question部分的包含的实体数量。
11. ANCOUNT：占16位（无符号）。指明Answer部分的包含的RR（Resource Record）数量。
12. NSCOUNT：占16位（无符号）。指明Authority部分的包含的RR（Resource Record）数量。
13. ARCOUNT：占16位（无符号）。指明Additional部分的包含的RR（Resource Record）数量。



#### Question部分

![](https://blog-1251252991.cos.ap-chengdu.myqcloud.com/2017-04-15%2003-08-09%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

1. QNAME：字节数不定，以0x00作为结束符。表示查询的主机名。注意：众所周知，主机名被"."号分割成了多段标签。在QNAME中，**每段标签前面加一个数字**，**表示接下来标签的长度**。比如：api.sina.com.cn表示成QNAME时，会在"api"前面加上一个字节0x03，"sina"前面加上一个字节0x04，"com"前面加上一个字节0x03，而"cn"前面加上一个字节0x02；

跟在quetion后面

1. QTYPE：占2个字节。表示RR类型，见以上RR介绍；
2. QCLASS：占2个字节。表示RR分类，见以上RR介绍。

### 二进制格式

05 baidu 03 com

### 



####  Answer、Authority、Additional部分

Answer、Authority、Additional部分**格式一致**，每部分都由若干实体组成，每个实体即为一条RR，之前有过介绍，格式如下图所示：

![](https://blog-1251252991.cos.ap-chengdu.myqcloud.com/2017-04-15%2003-11-32%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)



### DNS SRV



### K8s dns srv用于headless 普通 的就是A/AAAA



[Pod 与 Service 的 DNS | Kubernetes](https://kubernetes.io/zh/docs/concepts/services-networking/dns-pod-service/)

[DNS SRV介绍（一种用DNS做服务发现的方法）@小鸟技术笔记 (lijiaocn.com)](https://www.lijiaocn.com/技巧/2017/03/06/dns-srv.html)

SRV中除了记录**服务器**的地址，还记录了**服务**的**端口**，并且可以设置每个服务地址的优先级和权重。访问服务的时候，本地的DNS resolver从DNS服务器查询到一个地址列表，根据优先级和权重，从中选取一个地址作为本次请求的目标地址。



一个能够支持SRV的LDAP client可以通过查询域名，得知LDAP服务的IP地址和服务端口：

```
_ldap._tcp.example.com
```



SRV的DNS类型代码为33。

SRV的记录格式为：

```
_Service._Proto.Name TTL Class SRV Priority Weight Port Target

Service: 服务名称，前缀“_”是为防止与DNS Label（普通域名）冲突。
Proto:   服务使用的通信协议，_TCP、_UDP、其它标准协议或者自定义的协议。
Name:    提供服务的域名。
TTL:     缓存有效时间。
CLASS:   类别
Priority: 该记录的优先级，数值越小表示优先级越高，范围0-65535。
Weight:   该记录的权重，数值越高权重越高，范围0-65535。     
Port:     服务端口号，0-65535。
Target:   host地址。
```





### /etc/reslove.conf

[DNS for Services and Pods | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

[/etc/resolv.conf文件中的search项作用 - Loull - 博客园 (cnblogs.com)](https://www.cnblogs.com/549294286/p/9332555.html)



关键字

```
nameserver    //定义DNS服务器的IP地址

domain       //定义本地域名  声明主机的域名 很多程序用到它，如邮件系统；当为没有域名的主机进行DNS查询时，也要用到。如果没有域名，主机名将被使用，删除所有在第一个点( .)前面的内容。

search        //定义域名的搜索列表

sortlist        //对返回的域名进行排序
```





```sh
# cat /etc/resolv.conf
nameserver 192.168.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5



# search 定义域名的搜索列表，当查询的域名中包含的 . 的数量少于 options.ndots 的值时，会依次匹配列表中的每个值
```





