https://zhuanlan.zhihu.com/p/276147925

Quic 相比现在广泛应用的 http2+tcp+tls 协议有如下优势 [2]：

1. 减少了 TCP 三次握手及 TLS 握手时间。
2. 改进的拥塞控制。
3. 避免队头阻塞的多路复用。
4. 连接迁移。
5. 前向冗余纠错。





**改进的拥塞控制**

QUIC 协议当前默认使用了 TCP 协议的 Cubic 拥塞控制算法 [6]，同时也支持 CubicBytes, Reno, RenoBytes, BBR, PCC 等拥塞控制算法。

从拥塞算法本身来看，QUIC 只是按照 TCP 协议重新实现了一遍，那么 QUIC 协议到底改进在哪些方面呢？主要有如下几点：