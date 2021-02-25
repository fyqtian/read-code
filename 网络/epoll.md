### netpoller

https://zhuanlan.zhihu.com/p/299041493

https://blog.csdn.net/kezhi9195/article/details/92636074

https://zhuanlan.zhihu.com/p/299047984



https://blog.csdn.net/xxb249/article/details/105217956   epoll lt et



https://blog.csdn.net/xxb249/article/details/105234862 epoll EPOLLRDHUP

总结归纳了 5 种 I/O 模型，包括同步和异步 I/O：

- 阻塞 I/O (Blocking I/O)
- 非阻塞 I/O (Nonblocking I/O)
- I/O 多路复用 (I/O multiplexing)
- 信号驱动 I/O (Signal driven I/O)
- 异步 I/O (Asynchronous I/O)

首先我们先说一下什么是**文件描述符**（File descriptor），根据它的英文首字母也简称FD，它是一个用于表述指向文件的引用的抽象化概念。它是一个索引值，**指向内核为每一个进程所维护的该进程打开文件的记录表**。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。

操作系统上的 I/O 是用户空间和内核空间的数据交互，因此 I/O 操作通常包含以下两个步骤：

1. 等待网络数据到达网卡(读就绪)/等待网卡可写(写就绪) –> 读取/写入到内核缓冲区
2. 从内核缓冲区复制数据 –> 用户空间(读)/从用户空间复制数据 -> 内核缓冲区(写)



**而判定一个 I/O 模型是同步还是异步，主要看第二步**：数据在用户和内核空间之间复制的时候**是不是会阻塞当前进程**，如果会，则是同步 I/O，否则，就是异步 I/O。基于这个原则，这 5 种 I/O 模型中只有一种异步 I/O 模型：Asynchronous I/O，其余都是同步 I/O 模型。

![v2-618421e3b4dbb62cdd6acebab621534f_720w](D:\gocode\read-code\images\v2-618421e3b4dbb62cdd6acebab621534f_720w.jpg)



### **Non-blocking I/O**

什么叫非阻塞 I/O，顾名思义就是：**所有 I/O 操作都是立刻返回而不会阻塞当前用户进程**。**I/O 多路复用通常情况下需要和非阻塞 I/O 搭配使用**，否则可能会产生意想不到的问题。比如，**epoll 的 ET(边缘触发) 模式下**，如果不使用非阻塞 I/O，有极大的概率会导致阻塞 event-loop 线程，从而降低吞吐量，甚至导致 bug。

Linux 下，我们可以通过 `fcntl` 系统调用来设置 `O_NONBLOCK` 标志位，从而把 socket 设置成 Non-blocking。当对一个 Non-blocking socket 执行读操作时，流程是这个样子

当用户进程发出 read 操作时，**如果 kernel 中的数据还没有准备好，那么它并不会 block 用户进程**，而是立刻返回一个 **EAGAIN error**。从用户进程角度讲 ，它发起一个 read 操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个 error 时，它就知道数据还没有准备好，于是它可以再次发送 read 操作。一旦 kernel 中的数据准备好了，并且又再次收到了用户进程的 system call，那么它马上就将数据拷贝到了用户内存，然后返回。

![v2-ad0d52859325124245def7f9c0df430f_720w](D:\gocode\read-code\images\v2-ad0d52859325124245def7f9c0df430f_720w.jpg)

**所以，Non-blocking I/O 的特点是用户进程需要不断的主动询问 kernel 数据好了没有。下一节我们要讲的 I/O 多路复用需要和 Non-blocking I/O 配合才能发挥出最大的威力！**





### **I/O 多路复用**

所谓 I/O 多路复用指的就是 select/poll/epoll 这一系列的多路选择器：**支持单一线程同时监听多个文件描述符（I/O 事件）**，**阻塞等待，并在其中某个文件描述符可读写时收到通知**。 I/O 复用其实**复用的不是 I/O 连接，而是复用线程**，让一个 thread of control 能够处理多个连接（I/O 事件）。



### **select & poll**

```go
#include <sys/select.h>

/* According to earlier standards */
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

// 和 select 紧密结合的四个宏：
void FD_CLR(int fd, fd_set *set);
int FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);
```

select 是 epoll 之前 Linux 使用的 I/O 事件驱动技术。

理解 select 的关键在于理解 fd_set，为说明方便，取 fd_set 长度为 1 字节，fd_set 中的每一 bit 可以对应一个文件描述符 fd，则 1 字节长的 fd_set 最大可以对应 8 个 fd。select 的调用过程如下：

1. 执行 FD_ZERO(&set), 则 set 用位表示是 `0000,0000`
2. 若 fd＝5, 执行 FD_SET(fd, &set); 后 set 变为 0001,0000(第 5 位置为 1)
3. 再加入 fd＝2, fd=1，则 set 变为 `0001,0011`
4. 执行 select(6, &set, 0, 0, 0) 阻塞等待
5. 若 fd=1, fd=2 上都发生可读事件，则 select 返回，此时 set 变为 `0000,0011` (注意：没有事件发生的 fd=5 被清空)



#### 基于上面的调用过程，可以得出 select 的特点：

- **可监控的文件描述符个数取决于 sizeof(fd_set) 的值**。假设服务器上 sizeof(fd_set)＝512，每 bit 表示一个文件描述符，则服务器上支持的最大文件描述符是 512*8=4096。fd_set 的大小调整可参考 **[【原创】技术系列之 网络模型（二）](https://link.zhihu.com/?target=http%3A//www.cppblog.com/CppExplore/archive/2008/03/21/45061.html)** 中的模型 2，可以有效突破 select 可监控的文件描述符上限
- 将 fd 加入 select 监控集的同时，还要再使用一个数据结构 array 保存放到 select 监控集中的 fd，一是用于在 select 返回后，array 作为源数据和 fd_set 进行 FD_ISSET 判断。二是 select 返回后会把以前加入的但并无事件发生的 fd 清空，则每次开始 select 前都要重新从 array 取得 fd 逐一加入（FD_ZERO 最先），扫描 array 的同时取得 fd 最大值 maxfd，用于 select 的第一个参数
- 可见 select 模型必须在 select 前循环 array（加 fd，取 maxfd），select 返回后循环 array（FD_ISSET 判断是否有事件发生）



#### 所以，select 有如下的缺点：

1. 最大并发数限制：使用 32 个整数的 32 位，即 32*32=1024 来标识 fd，虽然可修改，但是有以下第 2, 3 点的瓶颈
2. 每次调用 select，**都需要把 fd 集合从用户态拷贝到内核态，这个开销在 fd 很多时会很大**
3. **性能衰减严重：每次 kernel 都需要线性扫描整个 fd_set，所以随着监控的描述符 fd 数量增长，**其 I/O 性能会线性下降



poll 的实现和 select 非常相似，**只是描述 fd 集合的方式不同**，poll 使用 pollfd 结构而不是 select 的 fd_set 结构，poll 解决了最大文件描述符数量限制的问题，**但是同样需要从用户态拷贝所有的 fd 到内核态，也需要线性遍历所有的 fd 集合**，所以它和 select 只是实现细节上的区分，并没有本质上的区别。



### **epoll**

![20190617154410853](..\images\20190617154410853.png)



![20190617154612872](..\images\20190617154612872.png)

![20190617154643670](..\images\20190617154643670.png)



![](..\images\20190617154711960.png)

![20190617154800768 (1)](..\images\20190617154800768.png)

![201906171549573](..\images\201906171549573.png)





其中，epoll_event的事件如下：



EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

   epoll在一个文件描述符上只能有一个事件，在一个描述符上添加多个事件，会产生EEXIST的错误。同样，删除epoll的事件，只需描述符就够了

   epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);

   这里有一个比较重要的问题：从epoll_wait返回的events中，该如何知道是哪个描述符上的事件：在注册epoll事件的时候，一定要填写epoll_data，否则我们将分不清触发的是哪个描述符上的事件。



\4. EPOLL事件有两种模型：

**Edge Triggered (ET) 边缘触发 只有数据到来，才触发，不管缓存区中是否还有数据。**
**Level Triggered (LT) 水平触发 只要有数据都会触发。**



epoll 是 Linux kernel 2.6 之后引入的新 I/O 事件驱动技术，**I/O 多路复用的核心设计是 1 个线程处理所有连接的 `等待消息准备好` I/O 事件**，这一点上 epoll 和 select&poll 是大同小异的。但 select&poll 错误预估了一件事，当数十万并发连接存在时，可能每一毫秒只有数百个活跃的连接，同时其余数十万连接在这一毫秒是非活跃的。select&poll 的使用方法是这样的： `返回的活跃连接 == select(全部待监控的连接)` 。

什么时候会调用 select&poll 呢？**在你认为需要找出有报文到达的活跃连接时**，就应该调用。所以，select&poll **在高并发时是会被频繁调用的**。这样，这个频繁调用的方法就很有必要看看它是否有效率，因为，它的轻微效率损失都会被 `高频` 二字所放大。它有效率损失吗？显而易见，全部待监控连接是数以十万计的，返回的只是数百个活跃连接，这本身就是无效率的表现。被放大后就会发现，处理并发上万个连接时，select&poll 就完全力不从心了。这个时候就该 epoll 上场了，epoll 通过一些新的设计和优化，基本上解决了 select&poll 的问题。



epoll 的 API 非常简洁，涉及到的只有 3 个系统调用：

```go
#include <sys/epoll.h>  
int epoll_create(int size); // int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

其中，

epoll_create 创建一个 epoll 实例并返回 epollfd；

epoll_ctl 注册 file descriptor 等待的 I/O 事件(比如 EPOLLIN、EPOLLOUT 等) 到 epoll 实例上；

epoll_wait 则是阻塞监听 epoll 实例上所有的 file descriptor 的 I/O 事件，**它接收一个用户空间上的一块内存地址 (events 数组)**，kernel 会在有 I/O 事件发生的时候把**文件描述符列表复制到这块内存地址上**，然后 epoll_wait 解除阻塞并返回，最后用户空间上的程序就可以对相应的 fd 进行读写了：

![v2-6697af18e4c3e16350529d15938adf6c_720w](D:\gocode\read-code\images\v2-6697af18e4c3e16350529d15938adf6c_720w.png)

与 select&poll 相比，epoll 分清了高频调用和低频调用。例如，epoll_ctl 相对来说就是非频繁调用的，**而 epoll_wait 则是会被高频调用的。所以 epoll 利用 epoll_ctl 来插入或者删除一个 fd**，**实现用户态到内核态的数据拷贝，这确保了每一个 fd 在其生命周期只需要被拷贝一次**，**而不是每次调用 epoll_wait 的时候都拷贝一次**。 epoll_wait 则被设计成几乎没有入参的调用，**相比 select&poll 需要把全部监听的 fd 集合从用户态拷贝至内核态的做法**，epoll 的效率就高出了一大截。

在实现上 epoll 采用红黑树来存储所有监听的 fd，而红黑树本身插入和删除性能比较稳定，时间复杂度 O(logN)。通过 epoll_ctl 函数添加进来的 fd 都会被放在红黑树的某个节点内，所以，重复添加是没有用的。当把 fd 添加进来的时候时候会完成关键的一步：该 fd 会与相应的设备（网卡）驱动程序建立回调关系，也就是在内核中断处理程序为它注册一个回调函数，在 fd 相应的事件触发（中断）之后（设备就绪了），内核就会调用这个回调函数，该回调函数在内核中被称为： `ep_poll_callback` ，**这个回调函数其实就是把这个 fd 添加到 rdllist 这个双向链表（就绪链表）中**。epoll_wait 实际上就是去检查 rdllist 双向链表中是否有就绪的 fd，当 rdllist 为空（无就绪 fd）时挂起当前进程，直到 rdllist 非空时进程才被唤醒并返回。

相比于 select&poll 调用时会将全部监听的 fd 从用户态空间拷贝至内核态空间并线性扫描一遍找出就绪的 fd 再返回到用户态，epoll_wait 则是直接返回已就绪 fd，因此 epoll 的 I/O 性能不会像 select&poll 那样随着监听的 fd 数量增加而出现线性衰减，是一个非常高效的 I/O 事件驱动技术。



由于使用 epoll 的 I/O 多路复用需要**用户进程自己负责 I/O 读写**，从用户进程的角度看，**读写过程是阻塞的**，所以 select&poll&epoll 本质上都是同步 I/O 模型，而像 Windows 的 **IOCP 这一类的异步 I/O，只需要在调用 WSARecv 或 WSASend 方法读写数据的时候把用户空间的内存 buffer 提交给 kernel**，kernel 负责数据在用户空间和内核空间拷贝，完成之后就会通知用户进程，整个过程不需要用户进程参与，所以是真正的异步 I/O。