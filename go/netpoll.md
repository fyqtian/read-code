![v2-bae882be10b54c89d50afcc8405b14d4_720w](..\images\v2-bae882be10b54c89d50afcc8405b14d4_720w.jpg)

### netpoller

https://zhuanlan.zhihu.com/p/299047984

https://strikefreedom.top/go-netpoll-io-multiplexing-reactor

https://www.cnblogs.com/luozhiyun/p/14390824.html

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/

Go netpoller 通过在底层对 epoll/kqueue/iocp 的封装，从而实现了使用同步编程模式达到异步执行的效果。总结来说，所有的网络操作都以网络描述符 netFD 为中心实现。netFD 与底层 PollDesc 结构绑定，当在一个 netFD 上读写遇到 EAGAIN 错误时，就将当前 goroutine 存储到这个 netFD 对应的 PollDesc 中，同时调用 gopark 把当前 goroutine 给 park 住，直到这个 netFD 上再次发生读写事件，才将此 goroutine 给 ready 激活重新运行。显然，在底层通知 goroutine 再次发生读写等事件的方式就是 epoll/kqueue/iocp 等事件驱动机制。



因为**文件 I/O**、网络 I/O 以及**计时器都依赖网络轮询器**，所以 Go 语言会通过以下两条不同路径初始化网络轮询器：

1. [`internal/poll.pollDesc.init`](https://draveness.me/golang/tree/internal/poll.pollDesc.init) — 通过 [`net.netFD.init`](https://draveness.me/golang/tree/net.netFD.init) 和 [`os.newFile`](https://draveness.me/golang/tree/os.newFile) 初始化网络 I/O 和文件 I/O 的轮询信息时；
2. [`runtime.doaddtimer`](https://draveness.me/golang/tree/runtime.doaddtimer) — 向处理器中增加新的计时器时；





- **`src/runtime/netpoll_epoll.go`**
- **`src/runtime/netpoll_kqueue.go`**
- **`src/runtime/netpoll_solaris.go`**
- **`src/runtime/netpoll_windows.go`**
- **`src/runtime/netpoll_aix.go`**
- **`src/runtime/netpoll_fake.go`**

不管是 Listener 的 Accept 还是 Conn 的 Read/Write 方法，都是基于一个 `netFD` 的数据结构的操作， `netFD` 是一个**网络描述符**，类似于 Linux 的文件描述符的概念，netFD 中包含一个 **poll.FD 数据结构**，而 poll.FD 中包含两个重要的数据结构 **Sysfd 和 pollDesc**，前者是**真正的系统文件描述符**，后者对是**底层事件驱动的封装**，所有的读写超时等操作都是通过调用后者的对应方法实现的。

调用链路

```
runtime.runtime_pollServerInit` --> `runtime.poll_runtime_pollServerInit` --> `runtime.netpollGenericInit

调用 epollcreate1 创建一个 epoll 实例 epfd，作为整个 runtime 的唯一 event-loop 使用；

调用 runtime.nonblockingPipe 创建一个用于和 epoll 实例通信的管道，这里为什么不用更新且更轻量的 eventfd 呢？我个人猜测是为了兼容更多以及更老的系统版本；

将 netpollBreakRd 通知信号量封装成 epollevent 事件结构体注册进 epoll 实例。
```

```go
net包
// Network file descriptor.
type netFD struct {
	pfd poll.FD

	// immutable until Close
	family      int
	sotype      int
	isConnected bool // handshake completed or use of association with peer
	net         string
	laddr       Addr
	raddr       Addr
}

func newFD(sysfd, family, sotype int, net string) (*netFD, error) {
   ret := &netFD{
      pfd: poll.FD{
         Sysfd:         sysfd,
         IsStream:      sotype == syscall.SOCK_STREAM,
         ZeroReadIsEOF: sotype != syscall.SOCK_DGRAM && sotype != syscall.SOCK_RAW,
      },
      family: family,
      sotype: sotype,
      net:    net,
   }
   return ret, nil
}

internal/poll
// FD is a file descriptor. The net and os packages embed this type in
// a larger type representing a network connection or OS file.
type FD struct {
	// Lock sysfd and serialize access to Read and Write methods.
	fdmu fdMutex

	// System file descriptor. Immutable until Close.
	Sysfd syscall.Handle

	// Read operation.
	rop operation
	// Write operation.
	wop operation

	// I/O poller.
	pd pollDesc

	// Used to implement pread/pwrite.
	l sync.Mutex

	// For console I/O.
	lastbits       []byte   // first few bytes of the last incomplete rune in last write
	readuint16     []uint16 // buffer to hold uint16s obtained with ReadConsole
	readbyte       []byte   // buffer to hold decoding of readuint16 from utf16 to utf8
	readbyteOffset int      // readbyte[readOffset:] is yet to be consumed with file.Read

	// Semaphore signaled when file is closed.
	csema uint32

	skipSyncNotif bool

	// Whether this is a streaming descriptor, as opposed to a
	// packet-based descriptor like a UDP socket.
	IsStream bool

	// Whether a zero byte read indicates EOF. This is false for a
	// message based socket connection.
	ZeroReadIsEOF bool

	// Whether this is a file rather than a network socket.
	isFile bool

	// The kind of this file.
	kind fileKind
}

internal/poll
type pollDesc struct {
	runtimeCtx uintptr    //实际是*runtime.pollDesc
}

//runtime pollDesc
type pollDesc struct {
	link *pollDesc // in pollcache, protected by pollcache.lock

	// The lock protects pollOpen, pollSetDeadline, pollUnblock and deadlineimpl operations.
	// This fully covers seq, rt and wt variables. fd is constant throughout the PollDesc lifetime.
	// pollReset, pollWait, pollWaitCanceled and runtime·netpollready (IO readiness notification)
	// proceed w/o taking the lock. So closing, everr, rg, rd, wg and wd are manipulated
	// in a lock-free way by all operations.
	// NOTE(dvyukov): the following code uses uintptr to store *g (rg/wg),
	// that will blow up when GC starts moving objects.
	lock    mutex // protects the following fields
	fd      uintptr
	closing bool
	everr   bool    // marks event scanning error happened
	user    uint32  // user settable cookie
	rseq    uintptr // protects from stale read timers
	rg      uintptr // pdReady, pdWait, G waiting for read or nil
	rt      timer   // read deadline timer (set if rt.f != nil)
	rd      int64   // read deadline
	wseq    uintptr // protects from stale write timers
	wg      uintptr // pdReady, pdWait, G waiting for write or nil
	wt      timer   // write deadline timer
	wd      int64   // write deadline
}
rseq 和 wseq — 表示文件描述符被重用或者计时器被重置5；
rg 和 wg — 表示二进制的信号量，可能为 pdReady、pdWait、等待文件描述符可读或者可写的 Goroutine 以及 nil；
rd 和 wd — 等待文件描述符可读或者可写的截止日期；
rt 和 wt — 用于等待文件描述符的计时器；

type pollCache struct {
   lock  mutex
   first *pollDesc
   // PollDesc objects must be type-stable,
   // because we can get ready notification from epoll/kqueue
   // after the descriptor is closed/reused.
   // Stale notifications are detected using seq variable,
   // seq is incremented when deadlines are changed or descriptor is reused.
}

var serverInit sync.Once

func (pd *pollDesc) init(fd *FD) error {
    //初始化runtime/poll
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		if ctx != 0 {
			runtime_pollUnblock(ctx)
			runtime_pollClose(ctx)
		}
		return errnoErr(syscall.Errno(errno))
	}
	pd.runtimeCtx = ctx
	return nil
}

//runtime
//go:linkname poll_runtime_pollServerInit internal/poll.runtime_pollServerInit
func poll_runtime_pollServerInit() {
	netpollGenericInit()
}

func netpollGenericInit() {
	if atomic.Load(&netpollInited) == 0 {
		lockInit(&netpollInitLock, lockRankNetpollInit)
		lock(&netpollInitLock)
		if netpollInited == 0 {
			netpollinit()
			atomic.Store(&netpollInited, 1)
		}
		unlock(&netpollInitLock)
	}
}
//初始化
func netpollinit() {
    //创建epollfd
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	if epfd < 0 {
		epfd = epollcreate(1024)
		if epfd < 0 {
			println("runtime: epollcreate failed with", -epfd)
			throw("runtime: netpollinit failed")
		}
		closeonexec(epfd)
	}
	r, w, errno := nonblockingPipe()
	if errno != 0 {
		println("runtime: pipe failed with", -errno)
		throw("runtime: pipe failed")
	}
	ev := epollevent{
		events: _EPOLLIN,   //读事件
	}
	*(**uintptr)(unsafe.Pointer(&ev.data)) = &netpollBreakRd
	errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)
	if errno != 0 {
		println("runtime: epollctl failed with", -errno)
		throw("runtime: epollctl failed")
	}
	netpollBreakRd = uintptr(r)
	netpollBreakWr = uintptr(w)
}


// netpollopen 会被 runtime_pollOpen 调用，注册 fd 到 epoll 实例，
// 注意这里使用的是 epoll 的 ET 模式，同时会利用万能指针把 pollDesc 保存到 epollevent 的一个 8 位的字节数组 data 里
func netpollopen(fd uintptr, pd *pollDesc) int32 {
 var ev epollevent
 ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
 *(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
 return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}

//go:linkname poll_runtime_pollOpen internal/poll.runtime_pollOpen
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
	pd := pollcache.alloc()
	lock(&pd.lock)
	if pd.wg != 0 && pd.wg != pdReady {
		throw("runtime: blocked write on free polldesc")
	}
	if pd.rg != 0 && pd.rg != pdReady {
		throw("runtime: blocked read on free polldesc")
	}
	pd.fd = fd
	pd.closing = false
	pd.everr = false
	pd.rseq++
	pd.rg = 0
	pd.rd = 0
	pd.wseq++
	pd.wg = 0
	pd.wd = 0
	unlock(&pd.lock)

	var errno int32
	errno = netpollopen(fd, pd)
	return pd, int(errno)
}


type pollCache struct {
	lock  mutex
	first *pollDesc
	// PollDesc objects must be type-stable,
	// because we can get ready notification from epoll/kqueue
	// after the descriptor is closed/reused.
	// Stale notifications are detected using seq variable,
	// seq is incremented when deadlines are changed or descriptor is reused.
}

func (c *pollCache) alloc() *pollDesc {
	lock(&c.lock)
	if c.first == nil {
		const pdSize = unsafe.Sizeof(pollDesc{})
		n := pollBlockSize / pdSize
		if n == 0 {
			n = 1
		}
		// Must be in non-GC memory because can be referenced
		// only from epoll/kqueue internals.
		mem := persistentalloc(n*pdSize, 0, &memstats.other_sys)
		for i := uintptr(0); i < n; i++ {
			pd := (*pollDesc)(add(mem, i*pdSize))
			pd.link = c.first
			c.first = pd
		}
	}
	pd := c.first
	c.first = pd.link
	lockInit(&pd.lock, lockRankPollDesc)
	unlock(&c.lock)
	return pd
}

```

### **Listener.Accept()**

`netpoll` accept socket 的工作流程如下：

- 服务端的 netFD 在 listen 时会创建 epoll 的实例，并将 listenerFD 加入 epoll 的事件队列
- netFD 在 accept 时将返回的 connFD 也加入 epoll 的事件队列
- netFD 在读写时出现 syscall.EAGAIN 错误，通过 pollDesc 的 waitRead 方法将当前的 goroutine park 住，直到 ready，从 pollDesc 的 waitRead 中返回

`Listener.Accept()` 接收来自客户端的新连接，具体还是调用 `netFD.accept` 方法来完成这个功能：



```go
// Accept implements the Accept method in the Listener interface; it
// waits for the next call and returns a generic Conn.
func (l *TCPListener) Accept() (Conn, error) {
 if !l.ok() {
  return nil, syscall.EINVAL
 }
 c, err := l.accept()
 if err != nil {
  return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
 }
 return c, nil
}

func (ln *TCPListener) accept() (*TCPConn, error) {
 fd, err := ln.fd.accept()
 if err != nil {
  return nil, err
 }
 tc := newTCPConn(fd)
 if ln.lc.KeepAlive >= 0 {
  setKeepAlive(fd, true)
  ka := ln.lc.KeepAlive
  if ln.lc.KeepAlive == 0 {
   ka = defaultTCPKeepAlive
  }
  setKeepAlivePeriod(fd, ka)
 }
 return tc, nil
}

func (fd *netFD) accept() (netfd *netFD, err error) {
 // 调用 poll.FD 的 Accept 方法接受新的 socket 连接，返回 socket 的 fd
 d, rsa, errcall, err := fd.pfd.Accept()
 if err != nil {
  if errcall != "" {
   err = wrapSyscallError(errcall, err)
  }
  return nil, err
 }
 // 以 socket fd 构造一个新的 netFD，代表这个新的 socket
 if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
  poll.CloseFunc(d)
  return nil, err
 }
 // 调用 netFD 的 init 方法完成初始化
 if err = netfd.init(); err != nil {
  fd.Close()
  return nil, err
 }
 lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
 netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
 return netfd, nil
}
```



`netFD.accept` 方法里会再调用 `poll.FD.Accept` ，最后会使用 Linux 的系统调用 `accept` 来完成新连接的接收，并且会把 accept 的 socket 设置成非阻塞 I/O 模式：

```go
// Accept wraps the accept network call.
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
 if err := fd.readLock(); err != nil {
  return -1, nil, "", err
 }
 defer fd.readUnlock()

 if err := fd.pd.prepareRead(fd.isFile); err != nil {
  return -1, nil, "", err
 }
 for {
  // 使用 linux 系统调用 accept 接收新连接，创建对应的 socket
  s, rsa, errcall, err := accept(fd.Sysfd)
  // 因为 listener fd 在创建的时候已经设置成非阻塞的了，
  // 所以 accept 方法会直接返回，不管有没有新连接到来；如果 err == nil 则表示正常建立新连接，直接返回
  if err == nil {
   return s, rsa, "", err
  }
  // 如果 err != nil，则判断 err == syscall.EAGAIN，符合条件则进入 pollDesc.waitRead 方法
  switch err {
  case syscall.EAGAIN:
   if fd.pd.pollable() {
    // 如果当前没有发生期待的 I/O 事件，那么 waitRead 会通过 park goroutine 让逻辑 block 在这里
    if err = fd.pd.waitRead(fd.isFile); err == nil {
     continue
    }
   }
  case syscall.ECONNABORTED:
   // This means that a socket on the listen
   // queue was closed before we Accept()ed it;
   // it's a silly error, so try again.
   continue
  }
  return -1, nil, errcall, err
 }
}

// 使用 linux 的 accept 系统调用接收新连接并把这个 socket fd 设置成非阻塞 I/O
ns, sa, err := Accept4Func(s, syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
// On Linux the accept4 system call was introduced in 2.6.28
// kernel and on FreeBSD it was introduced in 10 kernel. If we
// get an ENOSYS error on both Linux and FreeBSD, or EINVAL
// error on Linux, fall back to using accept.

// Accept4Func is used to hook the accept4 call.
var Accept4Func func(int, int) (int, syscall.Sockaddr, error) = syscall.Accept4
```



`pollDesc.waitRead` 方法主要负责检测当前这个 pollDesc 的上层 netFD 对应的 fd 是否有『期待的』I/O 事件发生，如果有就直接返回，否则就 park 住当前的 goroutine 并持续等待直至对应的 fd 上发生可读/可写或者其他『期待的』I/O 事件为止，然后它就会返回到外层的 for 循环，让 goroutine 继续执行逻辑。

poll.FD.Accept() 返回之后，会构造一个对应这个新 socket 的 netFD，然后调用 init() 方法完成初始化，这个 init 过程和前面 net.Listen() 是一样的，调用链：netFD.init() --> poll.FD.Init() --> poll.pollDesc.init()，最终又会走到这里：







### **Conn.Read/Conn.Write**

我们先来看看 `Conn.Read` 方法是如何实现的，原理其实和 `Listener.Accept` 是一样的，具体调用链还是首先调用 conn 的 `netFD.Read` ，然后内部再调用 `poll.FD.Read` ，最后使用 Linux 的系统调用 read: `syscall.Read` 完成数据读取：

```go
// Implementation of the Conn interface.

// Read implements the Conn Read method.
func (c *conn) Read(b []byte) (int, error) {
 if !c.ok() {
  return 0, syscall.EINVAL
 }
 n, err := c.fd.Read(b)
 if err != nil && err != io.EOF {
  err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
 }
 return n, err
}

func (fd *netFD) Read(p []byte) (n int, err error) {
 n, err = fd.pfd.Read(p)
 runtime.KeepAlive(fd)
 return n, wrapSyscallError("read", err)
}

// Read implements io.Reader.
func (fd *FD) Read(p []byte) (int, error) {
 if err := fd.readLock(); err != nil {
  return 0, err
 }
 defer fd.readUnlock()
 if len(p) == 0 {
  // If the caller wanted a zero byte read, return immediately
  // without trying (but after acquiring the readLock).
  // Otherwise syscall.Read returns 0, nil which looks like
  // io.EOF.
  // TODO(bradfitz): make it wait for readability? (Issue 15735)
  return 0, nil
 }
 if err := fd.pd.prepareRead(fd.isFile); err != nil {
  return 0, err
 }
 if fd.IsStream && len(p) > maxRW {
  p = p[:maxRW]
 }
 for {
  // 尝试从该 socket 读取数据，因为 socket 在被 listener accept 的时候设置成
  // 了非阻塞 I/O，所以这里同样也是直接返回，不管有没有可读的数据
  n, err := syscall.Read(fd.Sysfd, p)
  if err != nil {
   n = 0
   // err == syscall.EAGAIN 表示当前没有期待的 I/O 事件发生，也就是 socket 不可读
   if err == syscall.EAGAIN && fd.pd.pollable() {
    // 如果当前没有发生期待的 I/O 事件，那么 waitRead 
    // 会通过 park goroutine 让逻辑 block 在这里
    if err = fd.pd.waitRead(fd.isFile); err == nil {
     continue
    }
   }

   // On MacOS we can see EINTR here if the user
   // pressed ^Z.  See issue #22838.
   if runtime.GOOS == "darwin" && err == syscall.EINTR {
    continue
   }
  }
  err = fd.eofError(n, err)
  return n, err
 }
}
```

`conn.Write` 和 `conn.Read` 的原理是一致的，它也是通过类似 `pollDesc.waitRead` 的 `pollDesc.waitWrite` 来 park 住 goroutine 直至期待的 I/O 事件发生才返回恢复执行。



### **pollDesc.waitRead/pollDesc.waitWrite**

`pollDesc.waitRead` 内部调用了 `poll.runtime_pollWait` --> `runtime.poll_runtime_pollWait` 来达成无 I/O 事件时 park 住 goroutine 的目的：



```go
//go:linkname poll_runtime_pollWait internal/poll.runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
 err := netpollcheckerr(pd, int32(mode))
 if err != pollNoError {
  return err
 }
 // As for now only Solaris, illumos, and AIX use level-triggered IO.
 if GOOS == "solaris" || GOOS == "illumos" || GOOS == "aix" {
  netpollarm(pd, mode)
 }
 // 进入 netpollblock 并且判断是否有期待的 I/O 事件发生，
 // 这里的 for 循环是为了一直等到 io ready
 for !netpollblock(pd, int32(mode), false) {
  err = netpollcheckerr(pd, int32(mode))
  if err != 0 {
   return err
  }
  // Can happen if timeout has fired and unblocked us,
  // but before we had a chance to run, timeout has been reset.
  // Pretend it has not happened and retry.
 }
 return 0
}

// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
 // gpp 保存的是 goroutine 的数据结构 g，这里会根据 mode 的值决定是 rg 还是 wg，
  // 前面提到过，rg 和 wg 是用来保存等待 I/O 就绪的 gorouine 的，后面调用 gopark 之后，
  // 会把当前的 goroutine 的抽象数据结构 g 存入 gpp 这个指针，也就是 rg 或者 wg
 gpp := &pd.rg
 if mode == 'w' {
  gpp = &pd.wg
 }

 // set the gpp semaphore to WAIT
 // 这个 for 循环是为了等待 io ready 或者 io wait
 for {
  old := *gpp
  // gpp == pdReady 表示此时已有期待的 I/O 事件发生，
  // 可以直接返回 unblock 当前 goroutine 并执行响应的 I/O 操作
  if old == pdReady {
   *gpp = 0
   return true
  }
  if old != 0 {
   throw("runtime: double wait")
  }
  // 如果没有期待的 I/O 事件发生，则通过原子操作把 gpp 的值置为 pdWait 并退出 for 循环
  if atomic.Casuintptr(gpp, 0, pdWait) {
   break
  }
 }

 // need to recheck error states after setting gpp to WAIT
 // this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
 // do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
  
 // waitio 此时是 false，netpollcheckerr 方法会检查当前 pollDesc 对应的 fd 是否是正常的，
 // 通常来说  netpollcheckerr(pd, mode) == 0 是成立的，所以这里会执行 gopark 
 // 把当前 goroutine 给 park 住，直至对应的 fd 上发生可读/可写或者其他『期待的』I/O 事件为止，
 // 然后 unpark 返回，在 gopark 内部会把当前 goroutine 的抽象数据结构 g 存入
 // gpp(pollDesc.rg/pollDesc.wg) 指针里，以便在后面的 netpoll 函数取出 pollDesc 之后，
 // 把 g 添加到链表里返回，接着重新调度 goroutine
 if waitio || netpollcheckerr(pd, mode) == 0 {
  // 注册 netpollblockcommit 回调给 gopark，在 gopark 内部会执行它，保存当前 goroutine 到 gpp
  gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
 }
 // be careful to not lose concurrent READY notification
 old := atomic.Xchguintptr(gpp, 0)
 if old > pdWait {
  throw("runtime: corrupted polldesc")
 }
 return old == pdReady
}

// gopark 会停住当前的 goroutine 并且调用传递进来的回调函数 unlockf，从上面的源码我们可以知道这个函数是
// netpollblockcommit
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
 if reason != waitReasonSleep {
  checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
 }
 mp := acquirem()
 gp := mp.curg
 status := readgstatus(gp)
 if status != _Grunning && status != _Gscanrunning {
  throw("gopark: bad g status")
 }
 mp.waitlock = lock
 mp.waitunlockf = unlockf
 gp.waitreason = reason
 mp.waittraceev = traceEv
 mp.waittraceskip = traceskip
 releasem(mp)
 // can't do anything that might move the G between Ms here.
  // gopark 最终会调用 park_m，在这个函数内部会调用 unlockf，也就是 netpollblockcommit，
 // 然后会把当前的 goroutine，也就是 g 数据结构保存到 pollDesc 的 rg 或者 wg 指针里
 mcall(park_m)
}

// park continuation on g0.
func park_m(gp *g) {
 _g_ := getg()

 if trace.enabled {
  traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
 }

 casgstatus(gp, _Grunning, _Gwaiting)
 dropg()

 if fn := _g_.m.waitunlockf; fn != nil {
  // 调用 netpollblockcommit，把当前的 goroutine，
  // 也就是 g 数据结构保存到 pollDesc 的 rg 或者 wg 指针里
  ok := fn(gp, _g_.m.waitlock)
  _g_.m.waitunlockf = nil
  _g_.m.waitlock = nil
  if !ok {
   if trace.enabled {
    traceGoUnpark(gp, 2)
   }
   casgstatus(gp, _Gwaiting, _Grunnable)
   execute(gp, true) // Schedule it back, never returns.
  }
 }
 schedule()
}

// netpollblockcommit 在 gopark 函数里被调用
func netpollblockcommit(gp *g, gpp unsafe.Pointer) bool {
 // 通过原子操作把当前 goroutine 抽象的数据结构 g，也就是这里的参数 gp 存入 gpp 指针，
 // 此时 gpp 的值是 pollDesc 的 rg 或者 wg 指针
 r := atomic.Casuintptr((*uintptr)(gpp), pdWait, uintptr(unsafe.Pointer(gp)))
 if r {
  // Bump the count of goroutines waiting for the poller.
  // The scheduler uses this to decide whether to block
  // waiting for the poller if there is nothing else to do.
  atomic.Xadd(&netpollWaiters, 1)
 }
 return r
}
```





`runtime.netpoll` 的核心逻辑是：

1. 根据调用方的入参 delay，设置对应的调用 `epollwait` 的 timeout 值；
2. 调用 `epollwait` 等待发生了可读/可写事件的 fd；
3. 循环 `epollwait` 返回的事件列表，处理对应的事件类型， 组装可运行的 goroutine 链表并返回。





```go
schedule symon传入的都是0
// netpoll checks for ready network connections.
// Returns list of goroutines that become runnable.
// delay < 0: blocks indefinitely
// delay == 0: does not block, just polls
// delay > 0: block for up to that many nanoseconds
func netpoll(delay int64) gList {
   if epfd == -1 {
      return gList{}
   }
   var waitms int32
   if delay < 0 {
      waitms = -1
   } else if delay == 0 {
      waitms = 0
   } else if delay < 1e6 {
      waitms = 1
   } else if delay < 1e15 {
      waitms = int32(delay / 1e6)
   } else {
      // An arbitrary cap on how long to wait for a timer.
      // 1e9 ms == ~11.5 days.
      waitms = 1e9
   }
   var events [128]epollevent
retry:
    // 超时等待就绪的 fd 读写事件
   n := epollwait(epfd, &events[0], int32(len(events)), waitms)
   if n < 0 {
      if n != -_EINTR {
         println("runtime: epollwait on fd", epfd, "failed with", -n)
         throw("runtime: netpoll failed")
      }
      // If a timed sleep was interrupted, just return to
      // recalculate how long we should sleep now.
      if waitms > 0 {
         return gList{}
      }
      goto retry
   }
    // toRun 是一个 g 的链表，存储要恢复的 goroutines，最后返回给调用方
   var toRun gList
   for i := int32(0); i < n; i++ {
      ev := &events[i]
      if ev.events == 0 {
         continue
      }
	// Go scheduler 在调用 findrunnable() 寻找 goroutine 去执行的时候，
  // 在调用 netpoll 之时会检查当前是否有其他线程同步阻塞在 netpoll，
  // 若是，则调用 netpollBreak 来唤醒那个线程，避免它长时间阻塞
      if *(**uintptr)(unsafe.Pointer(&ev.data)) == &netpollBreakRd {
         if ev.events != _EPOLLIN {
            println("runtime: netpoll: break fd ready for", ev.events)
            throw("runtime: netpoll: break fd ready for something unexpected")
         }
         if delay != 0 {
            // netpollBreak could be picked up by a
            // nonblocking poll. Only read the byte
            // if blocking.
            var tmp [16]byte
            read(int32(netpollBreakRd), noescape(unsafe.Pointer(&tmp[0])), int32(len(tmp)))
            atomic.Store(&netpollWakeSig, 0)
         }
         continue
      }
	// 判断发生的事件类型，读类型或者写类型等，然后给 mode 复制相应的值，
    // mode 用来决定从 pollDesc 里的 rg 还是 wg 里取出 goroutine
      var mode int32
      if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
         mode += 'r'
      }
      if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
         mode += 'w'
      }
      if mode != 0 {
           // 取出保存在 epollevent 里的 pollDesc
         pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
         pd.everr = false
         if ev.events == _EPOLLERR {
            pd.everr = true
         }
         // 调用 netpollready，传入就绪 fd 的 pollDesc，
   		// 把 fd 对应的 goroutine 添加到链表 toRun 中
         netpollready(&toRun, pd, mode)
      }
   }
   return toRun
}

// netpollready 调用 netpollunblock 返回就绪 fd 对应的 goroutine 的抽象数据结构 g
func netpollready(toRun *gList, pd *pollDesc, mode int32) {
 var rg, wg *g
 if mode == 'r' || mode == 'r'+'w' {
  rg = netpollunblock(pd, 'r', true)
 }
 if mode == 'w' || mode == 'r'+'w' {
  wg = netpollunblock(pd, 'w', true)
 }
 if rg != nil {
  toRun.push(rg)
 }
 if wg != nil {
  toRun.push(wg)
 }
}

// netpollunblock 会依据传入的 mode 决定从 pollDesc 的 rg 或者 wg 取出当时 gopark 之时存入的
// goroutine 抽象数据结构 g 并返回
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
 // mode == 'r' 代表当时 gopark 是为了等待读事件，而 mode == 'w' 则代表是等待写事件
 gpp := &pd.rg
 if mode == 'w' {
  gpp = &pd.wg
 }

 for {
  // 取出 gpp 存储的 g
  old := *gpp
  if old == pdReady {
   return nil
  }
  if old == 0 && !ioready {
   // Only set READY for ioready. runtime_pollWait
   // will check for timeout/cancel before waiting.
   return nil
  }
  var new uintptr
  if ioready {
   new = pdReady
  }
  // 重置 pollDesc 的 rg 或者 wg
  if atomic.Casuintptr(gpp, old, new) {
      // 如果该 goroutine 还是必须等待，则返回 nil
   if old == pdWait {
    old = 0
   }
   // 通过万能指针还原成 g 并返回
   return (*g)(unsafe.Pointer(old))
  }
 }
}

// netpollBreak 往通信管道里写入信号去唤醒 epollwait
func netpollBreak() {
 // 通过 CAS 避免重复的唤醒信号被写入管道，
 // 从而减少系统调用并节省一些系统资源
 if atomic.Cas(&netpollWakeSig, 0, 1) {
  for {
   var b byte
   n := write(netpollBreakWr, unsafe.Pointer(&b), 1)
   if n == 1 {
    break
   }
   if n == -_EINTR {
    continue
   }
   if n == -_EAGAIN {
    return
   }
   println("runtime: netpollBreak write failed with", -n)
   throw("runtime: netpollBreak write failed")
  }
 }
}
```

Go 在多种场景下都可能会调用 `netpoll` 检查文件描述符状态，`netpoll` 里会调用 `epoll_wait` 从 epoll 的 `eventpoll.rdllist` 就绪双向链表返回，**从而得到 I/O 就绪的 socket fd 列表**，并根据取出最初调用 **`epoll_ctl` 时保存的上下文信息，恢复 `g**`。所以执行完`netpoll` 之后，会返回一个就绪 fd 列表对应的 goroutine 链表，接下来将就绪的 goroutine 通过调用 `injectglist` 加入到全局调度队列或者 P 的本地调度队列中，启动 M 绑定 P 去执行。

具体调用 `netpoll` 的地方，首先在 **Go runtime scheduler 循环调度 goroutines 之时就有可能会调用 `netpoll` 获取到已就绪的 fd 对应的 goroutine 来调度执行。**

首先 Go scheduler 的核心方法 `runtime.schedule()` 里会调用一个叫 `runtime.findrunable()` 的方法获取可运行的 goroutine 来执行，而在 `runtime.findrunable()` 方法里就调用了 `runtime.netpoll` 获取已就绪的 fd 列表对应的 goroutine 列表：



另外， `sysmon` 监控线程会在循环过程中检查距离上一次 `runtime.netpoll` **被调用是否超过了 10ms**，若是则会去调用它拿到可运行的 goroutine 列表并通过调用 injectglist 把 g 列表放入全局调度队列或者当前 P 本地调度队列等待被执行：











**网络轮询器和计时器的关系非常紧密**，这不仅仅是因为网络轮询器负责计时器的唤醒，还因为文件和网络 I/O 的截止日期也由网络轮询器负责处理。截止日期在 I/O 操作中，尤其是网络调用中很关键，网络请求存在很高的不确定因素，我们需要设置一个截止日期保证程序的正常运行，这时需要用到网络轮询器中的 [`runtime.poll_runtime_pollSetDeadline`](https://draveness.me/golang/tree/runtime.poll_runtime_pollSetDeadline)：



该函数会先使用截止日期计算出过期的时间点，然后根据 [`runtime.pollDesc`](https://draveness.me/golang/tree/runtime.pollDesc) 的状态做出以下不同的处理：

1. 如果结构体中的计时器没有设置执行的函数时，该函数会设置计时器到期后执行的函数、传入的参数并调用 [`runtime.resettimer`](https://draveness.me/golang/tree/runtime.resettimer) 重置计时器；
2. 如果结构体的读截止日期已经被改变，我们会根据新的截止日期做出不同的处理：
   1. 如果新的截止日期大于 0，调用 [`runtime.modtimer`](https://draveness.me/golang/tree/runtime.modtimer) 修改计时器；
   2. 如果新的截止日期小于 0，调用 [`runtime.deltimer`](https://draveness.me/golang/tree/runtime.deltimer) 删除计时器；

在 [`runtime.poll_runtime_pollSetDeadline`](https://draveness.me/golang/tree/runtime.poll_runtime_pollSetDeadline) 的最后，会重新检查轮询信息中存储的截止日期：



在 [`runtime.poll_runtime_pollSetDeadline`](https://draveness.me/golang/tree/runtime.poll_runtime_pollSetDeadline) 中直接调用 [`runtime.netpollgoready`](https://draveness.me/golang/tree/runtime.netpollgoready) 是相对比较特殊的情况。在正常情况下，运行时都会在计时器到期时调用 [`runtime.netpollDeadline`](https://draveness.me/golang/tree/runtime.netpollDeadline)、[`runtime.netpollReadDeadline`](https://draveness.me/golang/tree/runtime.netpollReadDeadline) 和 [`runtime.netpollWriteDeadline`](https://draveness.me/golang/tree/runtime.netpollWriteDeadline) 三个函数：

```go
//go:linkname poll_runtime_pollSetDeadline internal/poll.runtime_pollSetDeadline
func poll_runtime_pollSetDeadline(pd *pollDesc, d int64, mode int) {
   lock(&pd.lock)
   if pd.closing {
      unlock(&pd.lock)
      return
   }
   rd0, wd0 := pd.rd, pd.wd
   combo0 := rd0 > 0 && rd0 == wd0
   if d > 0 {
      d += nanotime()
      if d <= 0 {
         // If the user has a deadline in the future, but the delay calculation
         // overflows, then set the deadline to the maximum possible value.
         d = 1<<63 - 1
      }
   }
   if mode == 'r' || mode == 'r'+'w' {
      pd.rd = d
   }
   if mode == 'w' || mode == 'r'+'w' {
      pd.wd = d
   }
   combo := pd.rd > 0 && pd.rd == pd.wd
   rtf := netpollReadDeadline
   if combo {
      rtf = netpollDeadline
   }
   if pd.rt.f == nil {
      if pd.rd > 0 {
         pd.rt.f = rtf
         // Copy current seq into the timer arg.
         // Timer func will check the seq against current descriptor seq,
         // if they differ the descriptor was reused or timers were reset.
         pd.rt.arg = pd
         pd.rt.seq = pd.rseq
         resettimer(&pd.rt, pd.rd)
      }
   } else if pd.rd != rd0 || combo != combo0 {
      pd.rseq++ // invalidate current timers
      if pd.rd > 0 {
         modtimer(&pd.rt, pd.rd, 0, rtf, pd, pd.rseq)
      } else {
         deltimer(&pd.rt)
         pd.rt.f = nil
      }
   }
   if pd.wt.f == nil {
      if pd.wd > 0 && !combo {
         pd.wt.f = netpollWriteDeadline
         pd.wt.arg = pd
         pd.wt.seq = pd.wseq
         resettimer(&pd.wt, pd.wd)
      }
   } else if pd.wd != wd0 || combo != combo0 {
      pd.wseq++ // invalidate current timers
      if pd.wd > 0 && !combo {
         modtimer(&pd.wt, pd.wd, 0, netpollWriteDeadline, pd, pd.wseq)
      } else {
         deltimer(&pd.wt)
         pd.wt.f = nil
      }
   }
   // If we set the new deadline in the past, unblock currently pending IO if any.
   var rg, wg *g
   if pd.rd < 0 || pd.wd < 0 {
      atomic.StorepNoWB(noescape(unsafe.Pointer(&wg)), nil) // full memory barrier between stores to rd/wd and load of rg/wg in netpollunblock
      if pd.rd < 0 {
         rg = netpollunblock(pd, 'r', false)
      }
      if pd.wd < 0 {
         wg = netpollunblock(pd, 'w', false)
      }
   }
   unlock(&pd.lock)
   if rg != nil {
      netpollgoready(rg, 3)
   }
   if wg != nil {
      netpollgoready(wg, 3)
   }
}
```