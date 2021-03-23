### TIME_WAIT

根据第三版《UNIX网络编程 卷1》2.7节，TIME_WAIT状态的主要目的有两个：

- 优雅的关闭TCP连接，也就是尽量保证被动关闭的一端收到它自己发出去的FIN报文的ACK确认报文；
- 处理延迟的重复报文，这主要是为了避免前后两个使用相同四元组的连接中的前一个连接的报文干扰后一个连接。



如果只考虑上述第一个目标，则TIME_WAIT状态需要持续的时间应**该参考对端的RTO（重传超时时间）以及MSL（报文在网络中的最大生存时间）来计算而**不是仅仅按MSL来计算,因为只要对端没有收到针对FIN报文的ACK，就会一直持续重传FIN报文直到重传超时,所以最能实现完美关闭连接的时长计算方式应该是**从对端发送第一个FIN报文开始计时到它最后一次重传FIN报文这段时长加上MSL**,但这个计算方式过于保守，只有在所有的ACK报文都丢失的情况下才需要这么长的时间；另外，第一个目标虽然重要，但并不十分关键，因为既然已经到了关闭连接的最后一步，说明在这个TCP连接上的所有用户数据已经完成可靠传输，所以要不要完美的关闭这个连接其实已经不是那么关键了。



再来看一下《UNIX网络编程》在描述为什么需要TIME_WAIT状态时的一段话：

> Since the duration of the TIME_WAIT state is twice the MSL, this allows MSL seconds for packet in one direction to be lost, and another MSL seconds for the reply to be lost. By enforcing this rule, we are guaranteed that when we successfully establish a TCP connecton, all old duplicates from previous incarnations of the connection have expired in the network.



这段文字说明了TIME_WAIT状态持续2MSL的时间可以让一个TCP连接的两端发出的报文都从网络中消失，从而保证下一个使用了相同四元组的tcp连接不会被上一个连接的报文所干扰。



s

1. TCP连接中的一端发送了**FIN报文之后如果收不到对端针对该FIN的ACK，则会反复多次重传FIN报文**，大约持续几分钟；
2. 被动关闭处于LAST_ACK状态的一端在收到最后一个ACK之后不会发送任何报文，立即进入CLOSED状态；
3. 主动关闭的一端在收到被动关闭端发送过来**的FIN报文并回复ACK之后进入TIME_WAIT状态；**
4. 之所以TIME_WAIT状态需要维持一段时间而不是进入CLOSED状态，是因为需要处理对端可能重传的FIN报文或其它一些因网络原因而延迟的数据报文，不处理这些报文可能导致前后两个使用相同四元组的连接中的后一个连接出现异常(详见UNIX网络编程卷1的2.7节 第三版)；
5. 处于TIME_WAIT状态的一端在收到重传的FIN时会重新计时(rfc793 以及 linux kernel源代码tcp_timewait_state_process函数)。



<img src="..\images\6404.png" alt="6404" style="zoom:50%;" />



- 客户端打算关闭连接，此时会发送一个 TCP 首部 `FIN` 标志位被置为 `1` 的报文，也即 `FIN` 报文，之后客户端进入 `FIN_WAIT_1` 状态。
- 服务端收到该报文后，就向客户端发送 `ACK` 应答报文，接着服务端进入 `CLOSED_WAIT` 状态。
- 客户端收到服务端的 `ACK` 应答报文后，之后进入 `FIN_WAIT_2` 状态。
- 等待服务端处理完数据后，也向客户端发送 `FIN` 报文，之后服务端进入 `LAST_ACK` 状态。
- 客户端收到服务端的 `FIN` 报文后，回一个 `ACK` 应答报文，之后进入 `TIME_WAIT` 状态
- 服务器收到了 `ACK` 应答报文后，就进入了 `CLOSE` 状态，至此服务端已经完成连接的关闭。
- 客户端在经过 `2MSL` 一段时间后，自动进入 `CLOSE` 状态，至此客户端也完成连接的关闭。

你可以看到，每个方向都需要**一个 FIN 和一个 ACK**，因此通常被称为**四次挥手**。

这里一点需要注意是：**主动关闭连接的，才有 TIME_WAIT 状态。**



再来回顾下四次挥手双方发 `FIN` 包的过程，就能理解为什么需要四次了。

- 关闭连接时，客户端向服务端发送 `FIN` 时，**仅仅表示客户端不再发送数据了但是还能接收数据**。
- 服务器收到客户端的 `FIN` 报文时，先回一个 `ACK` 应答报文，**而服务端可能还有数据需要处理和发送，等服务端不再发送数据时**，才发送 `FIN` 报文给客户端来表示同意现在关闭连接。

从上面过程可知，**服务端通常需要等待完成数据的发送和处理，所以服务端的 `ACK` 和 `FIN` 一般都会分开发送**，从而比三次握手导致多了一次。



> ### 为什么 TIME_WAIT 等待的时间是 2MSL？

`MSL` 是 Maximum Segment Lifetime，**报文最大生存时间**，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。因为 TCP 报文基于是 IP 协议的，而 IP 头中有一个 `TTL` 字段，是 IP 数据报可以经过的最大路由数，每经过一个处理他的路由器此值就减 1，当此值为 0 则数据报将被丢弃，同时发送 ICMP 报文通知源主机。

MSL 与 TTL 的区别：**MSL 的单位是时间**，**而 TTL 是经过路由跳数**。所以 **MSL 应该要大于等于 TTL 消耗为 0 的时间**，以确保报文已被自然消亡。

TIME_WAIT 等待 2 倍的 MSL，比较合理的解释是：网络中可能存在来自发送方的数据包，当这些发送方的数据包被接收方处理后又会向对方发送响应，所以**一来一回需要等待 2 倍的时间**。

比如，如果被动关闭方没有收到断开连接的最后的 ACK 报文，就会触发超时重发 Fin 报文，另一方接收到 FIN 后，会重发 ACK 给被动关闭方， 一来一去正好 2 个 MSL。

`2MSL` 的时间是从**客户端接收到 FIN 后发送 ACK 开始计时的**。如果在 TIME-WAIT 时间内，因为客户端的 ACK 没有传输到服务端，客户端又接收到了服务端重发的 FIN 报文，那么 **2MSL 时间将重新计时**。

在 Linux 系统里 `2MSL` 默认是 `60` 秒，那么一个 `MSL` 也就是 `30` 秒。**Linux 系统停留在 TIME_WAIT 的时间为固定的 60 秒**。

其定义在 Linux 内核代码里的名称为 TCP_TIMEWAIT_LEN：



- **关于 MSL 和 TIME_WAIT**。通过上面的ISN的描述，相信你也知道MSL是怎么来的了。我们注意到，在TCP的状态图中，从TIME_WAIT状态到CLOSED状态，有一个超时设置，这个超时设置是 2*MSL（[RFC793](http://tools.ietf.org/html/rfc793)定义了MSL为2分钟，Linux设置成了30s）为什么要这有TIME_WAIT？为什么不直接给转成CLOSED状态呢？主要有两个原因：
- 1）TIME_WAIT确保有足够的时间让对端收到了ACK，如果被动关闭的那方没有收到Ack，就会触发被动端重发Fin，一来一去正好2个MSL，
- 2）有足够的时间让这个连接不会跟后面的连接混在一起（你要知道，有些自做主张的路由器会缓存IP数据包，如果连接被重用了，那么这些延迟收到的包就有可能会跟新连接混在一起）。你可以看看这篇文章《[TIME_WAIT and its design implications for protocols and scalable client server systems](http://www.serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html)》





#### 为什么需要 TIME_WAIT 状态？

主动发起关闭连接的一方，才会有 `TIME-WAIT` 状态。

需要 TIME-WAIT 状态，主要是两个原因：

- **防止具有相同「四元组」的「旧」数据包被收到；**
- 保证「被动关闭连接」的一方能被正确的关闭，**即保证最后的 ACK 能让被动关闭方接收，从而帮助其正常关闭；**



#### *原因一：防止旧连接的数据包*

#### <img src="..\images\644.webp" alt="644" style="zoom:67%;" />

- 如上图黄色框框服务端在关闭连接之前发送的 `SEQ = 301` 报文，被网络延迟了。
- 这时有相同端口的 TCP 连接被复用后，被延迟的 `SEQ = 301` 抵达了客户端，**那么客户端是有可能正常接收这个过期的报文，这就会产生数据错乱等严重的问题。**

**（最大作用 确认网络中这个4元组得包失效，不会引起错乱）**

所以，TCP 就设计出了这么一个机制，经过 `2MSL` 这个时间，**足以让两个方向上的数据包都被丢弃，使得原来连接的数据包在网络中都自然消失，再出现的数据包一定都是新建立连接所产生的。**





#### *原因二：保证连接正确关闭*

在 RFC 793 指出 TIME-WAIT 另一个重要的作用是：

*TIME-WAIT - represents waiting for enough time to pass to be sure the remote TCP received the acknowledgment of its connection termination request.*

也就是说，TIME-WAIT 作用是**等待足够的时间以确保最后的 ACK 能让被动关闭方接收，从而帮助其正常关闭。**

假设 TIME-WAIT 没有等待时间或时间过短，断开连接会造成什么问题呢？

![649](..\images\649.png)

- 如上图红色框框客户端四次挥手的最后一个 `ACK` 报文如果在网络中被丢失了，此时如果客户端 `TIME-WAIT` 过短或没有，则就直接进入了 `CLOSE` 状态了，那么服务端则会一直处在 `LASE-ACK` 状态。

- 当客户端发起建立连接的 `SYN` 请求报文后，服务端会发送 `RST` 报文给客户端，连接建立的过程就会被终止。

  **（相对来说还是因为当前4元组旧得包没消失得问题 ）**



如果 TIME-WAIT 等待足够长的情况就会遇到两种情况：

- 服务端正常收到四次挥手的最后一个 `ACK` 报文，则服务端正常关闭连接。
- 服务端没有收到四次挥手的最后一个 `ACK` 报文时，则会重发 `FIN` 关闭连接报文并等待新的 `ACK` 报文。

所以客户端在 `TIME-WAIT` 状态等待 `2MSL` 时间后，就可以**保证双方的连接都可以正常的关闭。**







如果服务器有处于 TIME-WAIT 状态的 TCP，**则说明是由服务器方主动发起的断开请求。**

过多的 TIME-WAIT 状态主要的危害有两种：

- **第一是内存资源占用**；
- 第二是对端口资源的占用，一个 TCP 连接至少消耗一个本地端口；

第二个危害是会造成严重的后果的，要知道，端口资源也是有限的，一般可以开启的端口为 `32768～61000`，也可以通过如下参数设置指定

```
net.ipv4.ip_local_port_range
```





**这里给出优化 TIME-WAIT 的几个方式，都是有利有弊：**

- 打开 net.ipv4.tcp_tw_reuse 和 net.ipv4.tcp_timestamps 选项；

- net.ipv4.tcp_max_tw_buckets

- 程序中使用 SO_LINGER ，应用强制使用 RST 关闭。

- tcp_tw_recycle

  



**关于tcp_tw_recycle**。如果是tcp_tw_recycle被打开了话，会假设对端开启了tcp_timestamps，然后会去比较时间戳，**如果时间戳变大了，就可以重用**。但是，如果对端是一个NAT网络的话（如：一个公司只用一个IP出公网）或是对端的IP被另一台重用了，这个事就复杂了。建链接的SYN可能就被直接丢掉了（你可能会看到connection time out的错误）（如果你想观摩一下Linux的内核代码，请参看源码[ tcp_timewait_state_process](http://lxr.free-electrons.com/ident?i=tcp_timewait_state_process)）。

**Again，使用tcp_tw_reuse和tcp_tw_recycle来解决TIME_WAIT的问题是非常非常危险的，因为这两个参数违反了TCP协议（[RFC 1122](http://tools.ietf.org/html/rfc1122)）**









如下的 Linux 内核参数开启后，则可以**复用处于 TIME_WAIT 的 socket 为新的连接所用**。

```
net.ipv4.tcp_tw_reuse = 1
使用这个选项，还有一个前提，需要打开对 TCP 时间戳的支持，即

net.ipv4.tcp_timestamps=1（默认即为 1）
这个时间戳的字段是在 TCP 头部的「选项」里，用于记录 TCP 发送方的当前时间戳和从对端接收到的最新时间戳。

由于引入了时间戳，我们在前面提到的 2MSL 问题就不复存在了，因为重复的数据包会因为时间戳过期被自然丢弃。
```

温馨提醒：`net.ipv4.tcp_tw_reuse`要慎用，因为使用了它就必然要打开时间戳的支持 `net.ipv4.tcp_timestamps`，**当客户端与服务端主机时间不同步时，客户端的发送的消息会被直接拒绝掉**。小林在工作中就遇到过。。。排查了非常的久

（没有全局时钟）



*方式二：net.ipv4.tcp_max_tw_buckets*

这个值默认为 18000，当系统中处于 TIME_WAIT 的连接**一旦超过这个值时，系统就会将所有的 TIME_WAIT 连接状态重置。**

这个方法过于暴力，而且治标不治本，带来的问题远比解决的问题多，**不推荐使用。**



*方式三：程序中使用 SO_LINGER*

我们可以通过设置 socket 选项，来设置调用 close 关闭连接行为。

```
struct linger so_linger;
so_linger.l_onoff = 1;
so_linger.l_linger = 0;
setsockopt(s, SOL_SOCKET, SO_LINGER, &so_linger,sizeof(so_linger));
```

如果`l_onoff`为非 0， 且`l_linger`值为 0，那么调用`close`后，**会立该发送一个`RST`标志给对端，该 TCP 连接将跳过四次挥手，也就跳过了`TIME_WAIT`状态，直接关闭。**

但这为跨越`TIME_WAIT`状态提供了一个可能，不过是一个非常危险的行为，不值得提倡。





int listen (int socketfd, int backlog)

在 Linux 内核 2.2 之后，backlog 变成 accept 队列，也就是已完成连接建立的队列长度，**所以现在通常认为 backlog 是 accept 队列。**