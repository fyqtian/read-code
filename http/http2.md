### HTTP2

https://www.zhihu.com/question/41609070 rpc优势

https://www.cnblogs.com/confach/p/10141273.html 

https://ye11ow.gitbooks.io/http2-explained/content/part2.html

http://httpwg.org/specs/rfc7541.html h2头压缩

https://httpwg.org/specs/rfc7540.html

http1.1现有问题

- 不能充分使用tcp资源，Tcp连接限制，http1.1的模型请求响应，单个链接不能同时发起多个http请求，在浏览器有请求限制的情况下（同一域名6-8个请求），需要使用多域名
- header内容较多，每次请求携带完成的header即使header没变化
- 明文传输
- 多张小图 需要合并到一张大图中 减少请求，或者base64
- js css类似多文件合并压缩（如果只需要其中一个文件 浪费带宽）

http2

- 请求优先级（priority帧）

- 流量控制（每条流？控制data帧）

- 头压缩（hpack）

- server-push（html响应后主动推送资源）

- 多路复用

- 二层分

- 重置（rst_stream帧）

  
  
  
  
  ### 概念
  
  **帧：**HTTP/2 数据通信的**最小单位消息**：指 HTTP/2 中逻辑上的 HTTP 消息。例如请求和响应等，**消息由一个或多个帧组成**。
  
  
  
  **流：**存在于连接中的一个虚拟通道。流可以承载双向消息，**每个流都有一个唯一的整数ID**。
  
  
  
  HTTP/2 采用**二进制格式传输数据**，而非 HTTP 1.x 的文本格式，二进制协议解析起来更高效。 HTTP / 1 的请求和响应报文，都是由起始行，首部和实体正文（可选）组成，各部分之间以**文本换行符分隔**。HTTP/2 将请求和响应数据分割为更小的帧，并且它们采用二进制编码。
  
  
  
  **HTTP/2 中，同域名下所有通信都在单个连接上完成，该连接可以承载任意数量的双向数据流。**每个数据流都以消息的形式发送，而消息又由一个或多个帧组成。**多个帧之间可以乱序发送**，根据帧**首部的流标识可以重新组装**。
  
  
  
  
  
  ### 头压缩
  
  简单说，HTTP头压缩需要在HTTP/2 Client和服务端之间：
  
  - 维护**一份相同的静态表（Static Table），包含常见的头部名称，以及特别常见的头部名称与值的组合**；
  - 维护一份**相同的动态表（Dynamic Table），可以动态地添加内容**；
  - 基于静态哈夫曼码表的哈夫曼编码（Huffman Coding）；
  
  在HTTP头里，有些key:value是固定，例如：
  
  ```bash
   :method: GET
   :scheme: http
  ```
  
  在编码时，它们直接用一个**index编号代替**，例如***:method:GET***是2，这些在一个静态表定义。静态表的定义如下，总共61个Header Name，点击URL[ https://tools.ietf.org/html/rfc7541#appendix-A](https://tools.ietf.org/html/rfc7541#appendix-A)查看所有静态表的定义。
  
  
  
  使用静态表、动态表、以及Huffman编码可以极大地提升压缩效果。对于静态表里的字段，原来需要N个字符表示的，现在只需要一个索引即可，对于静态、动态表中不存在的内容，还可以使用哈夫曼编码来减小体积。HTTP/2 标准里也给出了一份详细的静态哈夫曼码表（https://tools.ietf.org/html/rfc7541#appendix-B），它们需要内置在客户端和服务端之中。
  
  关于HPack的算法和实现，后面专门抽一篇文章来写。
  
  
  
  
  
  HTTP 1.1请求的大小变得越来越大，有时甚至会大于**TCP窗口的初始大小**，因为它们需要等待带着ACK的**响应（慢启动）**回来以后才能继续被发送。HTTP/2对消息头采用HPACK（专为http/2头部设计的压缩格式）进行压缩传输，能够节省消息头占用的网络的流量。而HTTP/1.x每次请求，都会携带大量冗余头信息，浪费了很多带宽资源。
  
  HTTP每一次通信都会携带一组头部，用于描述这次通信的的资源、浏览器属性、cookie等，例如
  
  <img src="..\images\v2-282eb9ed0d8dfd42f730784367dcf43d_720w.png" alt="v2-282eb9ed0d8dfd42f730784367dcf43d_720w" style="zoom:67%;" />
  
  
  
  
  
  为了减少这块的资源消耗并提升性能， **HTTP/2对这些首部采取了压缩策略**：
  
  - HTTP/2在**客户端和服务器端**使用“**首部表**”来跟踪和存储之前发送的键－值对，对于相同的数据，不再通过每次请求和响应发送；
  
  - **首部表在HTTP/2的连接存续期内始终存在**，由客户端和服务器共同**渐进地更新;**
  
  - 每个新的首部键－值对要么被追加到当前表的末尾，要么替换表中之前的值。
  
    **不太理解如果第一个请求get 第二个请求post 怎么处理首部状态**
  
  
  
  第一次请求
  
  <img src="..\images\v2-1b19808193ed42238410e88aedbd44fc_720w.jpg" alt="v2-1b19808193ed42238410e88aedbd44fc_720w" style="zoom:80%;" />
  
  第二次请求
  
  
  
  <img src="..\images\v2-0bc35f311f6cbc0110d81726ac71f98d_720w.jpg" alt="v2-0bc35f311f6cbc0110d81726ac71f98d_720w" style="zoom:80%;" />
  
  
  
  
  
  <img src="..\images\1249-20181221093723730-1731019507.png" alt="1249-20181221093723730-1731019507" style="zoom:67%;" />
  
  



## HTTP/2 ALPN

HTTP/2协议里有个negotiation的机制，让客户端和服务器选择使用**HTTP 1.1还是2.0**，这个是由**ALPN来实现**，关于ALPN，可以参看

ALPN（Transport Layer Security (TLS) Application-Layer Protocol Negotiation Extension，https://tools.ietf.org/html/rfc7301。 

下面是抓包截图，在TLS里的Client Hello的包里，我们可以看到ALPN里由H2和HTTP/1.1，这就是说客户端支持HTTP2以及HTTP 1.1.

<img src="..\images\1249-20181219102258373-891604425.png" alt="1249-20181219102258373-891604425" style="zoom:67%;" />