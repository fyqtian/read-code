traceroute



https://www.jianshu.com/p/802010d54849

https://www.cnblogs.com/zyd112/p/7196341.html

发送ttl递增的udp包

使用 UDP 的 traceroute，失败还是比较常见的。这常常是由于，在运营商的路由器上，**UDP 与 ICMP 的待遇大不相同**。为了利于 troubleshooting，ICMP ECHO Request/Reply 是不会封的，而 UDP 则不同。**UDP 常被用来做网络攻击，因为 UDP 无需连接，因而没有任何状态约束它，比较方便攻击者伪造源 IP、伪造目的端口发送任意多的 UDP 包，**长度自定义。所以运营商为安全考虑，对于 UDP 端口常常采用白名单 ACL，就是只有 ACL 允许的端口才可以通过，没有明确允许的则统统丢弃。比如允许 DNS/DHCP/SNMP 等。

traceroute -I ip 使用icmp

traceroute -T  -- port=port  ip 使用tcp 默认端口80



mtr

 https://www.cnblogs.com/xzkzzz/p/7413177.html

icmp包





https://www.cnblogs.com/iiiiher/p/8509506.html





pingplotter 收费 traceroute 14天