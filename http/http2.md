### HTTP2

https://www.zhihu.com/question/41609070 rpc优势

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

  