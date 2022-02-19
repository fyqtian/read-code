### tls

https://blog.helong.info/blog/2015/09/06/tls-protocol-analysis-and-crypto-protocol-design/

从上述分层的角度看，TLS大致是由3个组件拼成的：

- 对称加密传输组件，例如aes-128-gcm(这几个例子都是当前2015年最主流的选择);
- 认证密钥协商组件，例如rsa-ecdhe;
- 密钥扩展组件，例如TLS-PRF-sha256



这些组件可以再拆分为5类算法，在TLS中，这5类算法组合在一起，称为一个CipherSuite： authentication (认证算法) encryption (加密算法 ) message authentication code (消息认证码算法 简称MAC) key exchange (密钥交换算法) key derivation function （密钥衍生算法)

服务器端支持的CipherSuite列表，如果是用的openssl，可以用 openssl ciphers -V | column -t 命令查看，输出如：

<img src="..\images\cipher-suit.png" style="zoom:60%;" />

每一个CipherSuite分配有 一个2字节的数字用来标识 

表示： 名字为ECDHE-RSA-AES128-GCM-SHA256 的CipherSuite ，用于 TLSv1.2版本，使用 ECDHE 做密钥交换，使用RSA做认证，使用AES-128-gcm做加密算法，MAC由于gcm作为一种aead模式并不需要，所以显示为aead，使用SHA256做PRF算法。



### Record协议

record协议做应用数据的对称加密传输，占据一个TLS连接的绝大多数流量，因此，先看看record协议 图片来自网络:

<img src="..\images\tls-record.png" style="zoom:67%;" />

Record 协议 – 从应用层接受数据，并且做:

- 发送方分片，接收方重组
- 生成序列号，为每个数据块生成唯一编号，防止被重放或被重排序
- 压缩，可选步骤，使用握手协议协商出的压缩算法做压缩
- 加密，使用握手协议协商出来的key做加密/解密
- 算HMAC，对数据计算HMAC，并且验证收到的数据包的HMAC正确性
- 发给tcp/ip，把数据发送给 TCP/IP 做传输(或其它ipc机制)。





TLS协议规定：client发送ClientHello消息，server必须回复ServerHello消息，否则就是fatal error，当成连接失败处理。ClientHello和ServerHello消息用于建立client和server之间的安全增强能力，ClientHello和ServerHello消息建立如下属性：

- Protocol Version
- Session ID
- Cipher Suite
- Compression Method.

另外，产生并交换两个random值 ClientHello.random 和 ServerHello.random

密钥协商使用四条： server的Certificate，ServerKeyExchange，client的Certificate，ClientKeyExchange（可选） 。TLS规定以后如果要新增密钥协商方法，可以订制这4条消息的数据格式，并且指定这4条消息的使用方法。密钥协商得出的共享密钥**必须足够长**，**当前定义的密钥协商算法生成的密钥长度必须大于46字节**。





在hello消息之后，server会把自己的证书在一条Certificate消息里面发给客户端(如果需要做服务器端认证的话，例如https)。 并且，如果需要的话，server会发送一条ServerKeyExchange消息，（例如如果服务器的证书只用做签名，不用做密钥交换，或者服务器没有证书）。client对server的认证完成后，server可以要求client发送client的证书，如果这是协商出来的CipherSuite允许的。下一步，server会发送ServerHelloDone消息，表示握手的hello消息部分已经结束。然后server会等待一个client的响应。如果server已经发过了CertificateRequest消息，client必须发送Certificate消息。然后发送ClientKeyExchange消息，并且这条消息的内容取决于ClientHello和ServerHello消息协商的算法。如果client发送了有签名能力的证书，就显式发送一个经过数字签名的CertificateVerify消息，来证明自己拥有证书私钥。	







session 和ticket

https://blog.csdn.net/mrpre/article/details/77868669

http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html

### session id会话复用

  对于已经建立的SSL会话，**使用session id为key（session id来自第一次请求的server hello中的session id字段），主密钥为value组成一对键值，保存在本地，服务器和客户端都保存一份。**

  当第二次握手时，客户端若想使用会话复用，**则发起的client hello中session id会置上对应的值，**服务器收到这个client hello，解析session id，查找本地是否有该session id，如果有，判断当前的加密套件和上个会话的加密套件是否一致，一致则允许使用会话复用，于是自己的server hello 中session id也置上和client hello中一样的值。然后计算对称秘钥，解析后续的操作。

  如果服务器未查到客户端的session id指定的会话（可能是会话已经老化），则会重新握手，session id要么重新计算（和client hello中session id不一样），要么置成0，这两个方式都会告诉客户端这次会话不进行会话复用


**Session ticket的工作流程如下：**

1：客户端发起client hello，拓展中带上空的session ticket TLS，表明自己支持session ticket。

2：服务器在握手过程中，如果支持session ticket，**则发送New session ticket类型的握手报文，其中包含了能够恢复包括主密钥在内的会话信息，当然，最简单的就是只发送master key。为了让中间人不可见，这个session ticket部分会进行编码、加密等操作**。

3：客户端收到这个session ticket，**就把当前的master key和这个ticket组成一对键值保存起来**。服务器无需保存任何会话信息，客户端也无需知道session ticket具体表示什么。

4：当客户端尝试会话复用时，会在client hello的拓展中加上session ticket，然后服务器收到session ticket，回去进行解密、解码能相关操作，来恢复会话信息。如果能够恢复会话信息，那么久提取会话信息的主密钥进行后续的操作。



new session ticket 包含

- Lifetime Hint告知客户端session ticket老化时间，
- length 长度
- session ticket具体指由服务器计算，一般经过编码+加密流程，客户端无需知道具体内容。客户端收到后，将dip dport 与master key和 这个session ticket 关联起来。



重放攻击

直接重复发送流量包数据