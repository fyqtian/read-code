### http缓存

301 308 404也能被缓存





[HTTP 缓存机制详解_foolbird-CSDN博客_http缓存](https://blog.csdn.net/guduyibeizi/article/details/81814577)

## **强缓存**



不会向服务器发送请求，直接从**缓存中读取资源**，在chrome控制台的Network选项中可以看到该请求返回200的状态码，并且size显示**from disk cache或from memory cache两种**（灰色表示缓存）。



**Expires ：**response header里的过期时间，浏览器再次加载资源时，如果在这个过期时间内，则命中强缓存。

**Cache-Control:**当值设为max-age=300时，则代表在这个请求正确返回时间（浏览器也会记录下来）的5分钟内再次加载资源，就会命中强缓存。



区别：Expires 是http1.0的产物，**Cache-Control是http1.1的产物,**两者同时存在的话，Cache-Control优先级高于Expires

Expires其实是**过时的产物**，现阶段它的存在只是一种**兼容性的写法**







## **协商缓存**

向服务器发送请求，服务器会根据这个请求的**request header的一些参数来判断是否命中协商缓存**，如果命中，则返回304状态码并带上新的response header通知浏览器从缓存中读取资源；



- **ETag和If-None-Match:**

Etag是上一次加载资源时，服务器返回的**response header**，是对该资源的一种**唯一标识**，**只要资源有变化**，Etag就会重新生成，

浏览器在下一次加载资源向服务器发送请求时，会将**上一次返回的Etag**值放到**request header**里的**If-None-Match**里

服务器接受到If-None-Match的值后，会拿来跟该资源文件的Etag值**做比较**，如果相同，则表示资源文件没有发生改变，命中协商缓存。



- **Last-Modified和If-Modified-Since**

  Last-Modified是该资源文件**最后一次更改时间**,服务器会在**response header**里返回，同时浏览器会将这个值保存起来，下一次发送请求时，放到**request headr**里的**If-Modified-Since**里



在精确度上，**Etag要优于Last-Modified**，Last-Modified的时间单位是秒，如果某个文件在1秒内改变了多次，那么他们的Last-Modified其实并没有体现出来修改，但是Etag每次都会改变确保了精度



**在性能上**，Etag要逊于Last-Modified，毕竟Last-Modified只需要记录时间，而Etag需要服务器通过算法来计算出一个hash值。







**强缓存 VS 协商缓存：**最好是配合在一起用，争取最大化的减少请求，利用缓存，节约流量。









## **浏览器缓存过程：**

1. 浏览器**第一次**加载资源，服务器返回200，浏览器将资源文件从服务器上请求下载下来，并把**response header**及该请求的**返回时间**(要与Cache-Control和Expires对比)一并缓存；
2. 下一次加载资源时，先比较当前时间和上一次返回200时的**时间差**，如果没有超过Cache-Control设置的max-age，则没有过期，命中强缓存，不发请求直接从本地缓存读取该文件（如果浏览器不支持HTTP1.1，则用Expires判断是否过期）；
3. **如果时间过期**，服务器则查看header里的**If-None-Match**和**If-Modified-Since** ；
4. 服务器**优先根据Etag**的值判断被请求的文件有没有做修改，Etag值一致则没有修改，命中协商缓存，返回304；如果不一致则有改动，直接返回新的资源文件带上新的Etag值并返回 200；
5. 如果服务器收到的请求没有Etag值，则将**If-Modified-Since**和被请求文件的最后修改时间做比对，一致则命中协商缓存，返回304；不一致则返回新的**last-modified和文件**并返回 200；



**Cache-Control其他字段：**

- no-cache: 虽然字面意义是“不要缓存”。但它实际上的机制是，仍然对资源使用缓存，但每一次在使用缓存之前必须向服务器对缓存资源进行验证。
- no-store: 不使用任何缓存

```http
Cache-Control: no-cache, no-store, must-revalidate
```



Cache-Control: **public** 

表示一些中间代理、CDN等可以缓存资源，即便是带有一些敏感 HTTP 验证身份信息甚至响应状态代码通常无法缓存的也可以缓存。通常 public 是非必须的，因为响应头 max-age 信息已经明确告知可以缓存了。



Cache-Control: **private** 

明确告知此资源只能单个用户可以缓存，其他中间代理不能缓存。原始发起的浏览器可以缓存，中间代理不能缓存。例如：百度搜索时，特定搜索信息只能被发起请求的浏览器缓存。
