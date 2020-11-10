### HTTP2

http1.1现有问题

- Tcp连接限制，http1.1的模型请求响应，单个链接不能同时发起多个http请求，在浏览器有请求限制的情况下（同一域名6-8个请求），需要使用多域名
- header内容较多，每次请求携带完成的header即使header没变化
- 明文传输



http2

- 请求优先级
- 流量控制
- header压缩
- server-push
- 多路复用
- 二层分帧