### TIME_WAIT

根据第三版《UNIX网络编程 卷1》2.7节，TIME_WAIT状态的主要目的有两个：

- 优雅的关闭TCP连接，也就是尽量保证被动关闭的一端收到它自己发出去的FIN报文的ACK确认报文；
- 处理延迟的重复报文，这主要是为了避免前后两个使用相同四元组的连接中的前一个连接的报文干扰后一个连接。



如果只考虑上述第一个目标，则TIME_WAIT状态需要持续的时间应**该参考对端的RTO（重传超时时间）以及MSL（报文在网络中的最大生存时间）来计算而**不是仅仅按MSL来计算,因为只要对端没有收到针对FIN报文的ACK，就会一直持续重传FIN报文直到重传超时,所以最能实现完美关闭连接的时长计算方式应该是**从对端发送第一个FIN报文开始计时到它最后一次重传FIN报文这段时长加上MSL**,但这个计算方式过于保守，只有在所有的ACK报文都丢失的情况下才需要这么长的时间；另外，第一个目标虽然重要，但并不十分关键，因为既然已经到了关闭连接的最后一步，说明在这个TCP连接上的所有用户数据已经完成可靠传输，所以要不要完美的关闭这个连接其实已经不是那么关键了。



再来看一下《UNIX网络编程》在描述为什么需要TIME_WAIT状态时的一段话：

> Since the duration of the TIME_WAIT state is twice the MSL, this allows MSL seconds for packet in one direction to be lost, and another MSL seconds for the reply to be lost. By enforcing this rule, we are guaranteed that when we successfully establish a TCP connecton, all old duplicates from previous incarnations of the connection have expired in the network.



这段文字说明了TIME_WAIT状态持续2MSL的时间可以让一个TCP连接的两端发出的报文都从网络中消失，从而保证下一个使用了相同四元组的tcp连接不会被上一个连接的报文所干扰。





1. TCP连接中的一端发送了**FIN报文之后如果收不到对端针对该FIN的ACK，则会反复多次重传FIN报文**，大约持续几分钟；
2. 被动关闭处于LAST_ACK状态的一端在收到最后一个ACK之后不会发送任何报文，立即进入CLOSED状态；
3. 主动关闭的一端在收到被动关闭端发送过来**的FIN报文并回复ACK之后进入TIME_WAIT状态；**
4. 之所以TIME_WAIT状态需要维持一段时间而不是进入CLOSED状态，是因为需要处理对端可能重传的FIN报文或其它一些因网络原因而延迟的数据报文，不处理这些报文可能导致前后两个使用相同四元组的连接中的后一个连接出现异常(详见UNIX网络编程卷1的2.7节 第三版)；
5. 处于TIME_WAIT状态的一端在收到重传的FIN时会重新计时(rfc793 以及 linux kernel源代码tcp_timewait_state_process函数)。

