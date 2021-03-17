### http

https://draveness.me/golang/docs/part4-advanced/ch09-stdlib/golang-net-http/

作为文本传输协议，HTTP 协议的协议头都是**文本数据**，HTTP 请求头的**首行会包含请求的方法**、路径和协议版本，接下来是多个 HTTP 协议头以及携带的负载。



HTTP 响应也有着比较类似的结构，其中也包含响应的协议版本、状态码、响应头以及负载，在这里就不展开介绍了。

```http
GET / HTTP/1.1
User-Agent: Mozilla/4.0 (compatible; MSIE5.01; Windows NT)
Host: draveness.me
Accept-Language: en-us
Accept-Encoding: gzip, deflate
Content-Length: <length>
Connection: Keep-Alive

<html>
    ...
</html>
```



### 消息边界

HTTP 协议目前主要还是跑在 TCP 协议上的，TCP 协议是面向连接的、可靠的、基于字节流的传输层通信协议,应用层交给 TCP 协议的数据并不会以消息为单位向目的主机传输，这些数据在某些情况下会被组合成一个数据段发送给目标的主机。因为 TCP 协议是基于字节流的，所以基于 TCP 协议的应用层协议都需要自己划分消息的边界。



**图 9-7 实现消息边界的方法**

在应用层协议中，最常见的两种解决方案是基于**长度**或者基于**终结符**（Delimiter）(**定长，基本没见过**)。HTTP 协议其实**同时实现了上述两种方案**，在多数情况下 HTTP 协议都会在协议头中加入 `Content-Length` 表示负载的长度，消息的接收者解析到该协议头之后就可以确定当前 HTTP 请求/响应结束的位置，分离不同的 HTTP 消息，下面就是一个使用 `Content-Length` 划分消息边界的例子：

```html
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 138
...
Connection: close

<html>
  <head>
    <title>An Example Page</title>
  </head>
  <body>
    <p>Hello World, this is a very simple HTML document.</p>
  </body>
</html>
```





#### 分块传输

不过 HTTP 协议除了使用**基于长度**的方式实现边界，也会使用基于终结符的策略，当 HTTP **使用块传输（Chunked Transfer）机制时**，HTTP 头中就不再包含 `Content-Length` 了，它会使用负载**大小为 0 的 HTTP 消息作为终结符**表示消息的边界。



当客户端向服务器请求一个静态页面或者一张图片时，服务器可以很清楚的知道内容大小，然后通过Content-Length消息首部字段告诉客户端需要接收多少数据。但是如果是动态页面等时，服务器是不可能预先知道内容大小，这时就可以使用Transfer-Encoding：chunk模式来传输数据了。即如果要一边产生数据，一边发给客户端，服务器就需要使用"Transfer-Encoding: chunked"这样的方式来代替Content-Length。

由一个标明**长度为0的chunk结束**。每个chunk有**两部分组成**，第一部分是该chunk的长度，第二部分就是指定长度的内容，每个部分用CRLF隔开。在最后一个长度为0的chunk中的内容是称为footer的内容，是一些没有写的头部内容。

chunk编码格式如下：

(chunk size) (\r\n) (chunk data) (\r\n) (chunk size) (\r\n) (chunk data) (\r\n) (chunk size = 0) (\r\n)(\r\n)

chunk size是以十六进制的ASCII码表示，比如：头部是3134这两个字节，表示的是1和4这两个ascii字符，被http协议解释为十六进制数14，也就是十进制的20，后面紧跟[\r\n](0d 0a)，再接着是连续的20个字节的chunk正文。chunk数据以0长度的chunk块结束，也就是（30 0d 0a 0d 0a）。


在进行chunked编码传输时，在回复消息的头部有Transfer-Encoding: chunked

![8100269-64c076b39a125257](..\images\8100269-64c076b39a125257.webp)