### SO_REUSEPORT

https://mp.weixin.qq.com/s/BuZwjRNwuKx19PU6E95Oeg

http://www.blogjava.net/yongboy/archive/2015/02/12/422893.html



运行在Linux系统上网络应用程序，为了利用多核的优势，一般使用以下比较典型的多进程/多线程服务器模型：

1. **单线程listen/accept，多个工作线程接收任务分发**，虽CPU的工作负载不再是问题，但会存在：
   - 单线程listener，在处理高速率海量连接时，一样会成为瓶颈
   - CPU缓存行丢失套接字结构(socket structure)现象严重
2. **所有工作线程都accept()在同一个服务器套接字上呢**，一样存在问题：
   - 多线程访问server socket锁竞争严重
   - 高负载下，线程之间处理不均衡，有时高达3:1不均衡比例
   - 导致CPU缓存行跳跃(cache line bouncing)
   - 在繁忙CPU上存在较大延迟



Linux kernel 3.9带来了SO_REUSEPORT特性，可以解决以上大部分问题。

### SO_REUSEPORT解决了什么问题

linux man文档中一段文字描述其作用：

> The new socket option allows multiple sockets on the same host to bind to the same port, and is intended to improve the performance of multithreaded network server applications running on top of multicore systems.

SO_REUSEPORT支持多**个进程或者线程绑定到同一端口**，提高服务器程序的性能，解决的问题：

- 允许多个套接字 bind()/listen() 同一个TCP/UDP端口
  - 每一个线程拥有自己的服务器套接字
  - 在服务器套接字上没有了锁的竞争
- **内核层面实现负载均衡**
- 安全层面，监听同一个端口的套接字只能位于同一个用户下面

其核心的实现主要有三点：

- 扩展 socket option，增加 SO_REUSEPORT 选项，用来设置 reuseport。
- 修改 bind 系统调用实现，以便支持可以绑定到相同的 IP 和端口
- 修改处理新建连接的实现，查找 listener 的时候，能够支持在监听相同 IP 和端口的多个 sock 之间均衡选择。



以前通过**`fork`形式创建多个子进程**，现在有了SO_REUSEPORT，可以不用通过`fork`的形式，让多进程监听同一个端口，各个进程中`accept socket fd`不一样，**有新连接建立时，内核只会唤醒一个进程来`accept`，并且保证唤醒的均衡性。**

模型简单，维护方便了，进程的管理和应用逻辑解耦，进程的管理水平扩展权限下放给程序员/管理员，可以根据实际进行控制进程启动/关闭，增加了灵活性。

这带来了一个较为微观的水平扩展思路，线程多少是否合适，状态是否存在共享，降低单个进程的资源依赖，针对无状态的服务器架构最为适合了。

#### 服务器无缝重启/切换

想法是，我们迭代了一版本，需要部署到线上，为之启动一个新的进程后，稍后关闭旧版本进程程序，服务一直在运行中不间断，需要平衡过度。这就像Erlang语言层面所提供的热更新一样。

想法不错，但是实际操作起来，就不是那么平滑了，还好有一个[hubtime](https://github.com/amscanne/huptime)开源工具，原理为`SIGHUP信号处理器+SO_REUSEPORT+LD_RELOAD`，可以帮助我们轻松做到，有需要的同学可以检出试用一下。



### SO_REUSEPORT已知问题

SO_REUSEPORT根据数据包的四元组{src ip, src port, dst ip, dst port}和**当前绑定同一个端口的服务器套接字数量进行数据包分发**。若服务器套接字数量产生变化，内核会把本该上一个服务器套接字所处理的客户端连接所发送的数据包（比如三次握手期间的半连接，以及已经完成握手但在队列中排队的连接）分发到其它的服务器套接字上面，可能会导致客户端请求失败，一般可以使用：

- 使用固定的服务器套接字数量，不要在负载繁忙期间轻易变化
- 允许多个服务器套接字共享TCP请求表(Tcp request table)
- 不使用四元组作为Hash值进行选择本地套接字处理，挑选隶属于同一个CPU的套接字

与RFS/RPS/XPS-mq协作，可以获得进一步的性能：

- 服务器线程绑定到CPUs
- RPS分发TCP SYN包到对应CPU核上
- TCP连接被已绑定到CPU上的线程accept()
- XPS-mq(Transmit Packet Steering for multiqueue)，传输队列和CPU绑定，发送数据
- RFS/RPS保证同一个连接后续数据包都会被分发到同一个CPU上
- 网卡接收队列已经绑定到CPU，则RFS/RPS则无须设置
- 需要注意硬件支持与否

目的嘛，数据包的软硬中断、接收、处理等在一个CPU核上，并行化处理，尽可能做到资源利用最大化。



```go
https://github.com/gogf/greuse


// We can create two processes with this code.
// Do some requests, then watch the output of the console.
func main() {
   listener, err := greuse.Listen("tcp", ":8881")
   if err != nil {
      panic(err)
   }
   defer listener.Close()

   server := &http.Server{}
   http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
      fmt.Fprintf(w, "gid: %d, pid: %d\n", os.Getgid(), os.Getpid())
   })

   panic(server.Serve(listener))
}
```