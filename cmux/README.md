





### cmux在一个端口监听多种请求http,grpc,tcp,ws

看etcd源码的时候发现使用了这个类库，在同一端口实现多个协议服务，具体看下实现原理

项目地址https://github.com/soheilhy/cmux  tag v0.1.4

```go
//example
package main

import (
	"fmt"
	"github.com/soheilhy/cmux"
	"log"
	"net"
	"net/http"
)

func main() {
	l, err := net.Listen("tcp", ":23456")
	if err != nil {
		log.Fatal(err)
	}
	m := cmux.New(l)
	//注册http适配器
	httpL := m.Match(cmux.HTTP1Fast())
	go serveHTTP(httpL)
	//注册tcp适配器 必须放最后
	trpcL := m.Match(cmux.Any()) // Any means anything that is not yet matched.
	go serveRPC(trpcL)
	//流量入口
	m.Serve()
}

type exampleHTTPHandler struct{}
func (h *exampleHTTPHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "example http response")
}
func serveHTTP(l net.Listener) {
	s := &http.Server{
		Handler: &exampleHTTPHandler{},
	}
	if err := s.Serve(l); err != cmux.ErrListenerClosed {
		panic(err)
	}
}

func serveRPC(l net.Listener) {
	for {
		conn, err := l.Accept()
		if err != nil {
			if err != cmux.ErrListenerClosed {
				panic(err)
			}
			return
		}
		go func(c net.Conn) {
			for {
				b := make([]byte, 1024)
				n, err := c.Read(b)
				fmt.Println(n, err)
				if err != nil {
					return
				}
				fmt.Println(string(b[:n]))
			}
		}(conn)
	}
}
```



源码分析 主要逻辑在cmux.go

```
// Matcher 判断流属于哪一个协议
type Matcher func(io.Reader) bool

// MatchWriter 对Matcher包了一层
type MatchWriter func(io.Writer, io.Reader) bool

// CMux is a multiplexer for network connections.
type CMux interface {
	// Match 返回一个 net.Listener 用于查看（接受）至少匹配中一个匹配器的网络连接
    // 根据传入的顺序决定匹配器的优先级
	Match(...Matcher) net.Listener
	//返回一个net.Listner 根据传入的顺序决定匹配器的优先级
	MatchWithWriters(...MatchWriter) net.Listener
	//阻塞开启多路复用
	Serve() error
	HandleError(ErrorHandler)
	SetReadTimeout(time.Duration)
}

type cMux struct {
	root        net.Listener
	bufLen      int
	errh        ErrorHandler
	donec       chan struct{}
	sls         []matchersListener
	readTimeout time.Duration
}

//多路复用实现结构
func New(l net.Listener) CMux {
	return &cMux{
		root:        l,
		bufLen:      1024,
		errh:        func(_ error) bool { return true },
		donec:       make(chan struct{}),
		readTimeout: noTimeout,
	}
}
```



CMux.Server() 开始端口监听

```
unc (m *cMux) Serve() error {
	var wg sync.WaitGroup

	...

	for {
		//实际的net.Listener.Accept()
		c, err := m.root.Accept()
		if err != nil {
			if !m.handleErr(err) {
				return err
			}
			continue
		}

		wg.Add(1)
		//解析流
		go m.serve(c, m.donec, &wg)
	}
}

func (m *cMux) serve(c net.Conn, donec <-chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()
	//对conn进行封装 主要是处理从conn读出来的byte复用
	muc := newMuxConn(c)
	...
	//遍历注册的服务
	for _, sl := range m.sls {
		//遍历服务的matcher
		for _, s := range sl.ss {
			matched := s(muc.Conn, muc.startSniffing())
			//如果匹配成功
			if matched {
				//reset bytes
				muc.doneSniffing()
				if m.readTimeout > noTimeout {
					_ = c.SetReadDeadline(time.Time{})
				}
				select {
				//conn send到对应的服务
				case sl.l.connc <- muc:
				case <-donec:
					_ = c.Close()
				}
				return
			}
		}
	}
	...
}


```





```
//http

httpL := m.Match(cmux.HTTP1Fast())

func (m *cMux) Match(matchers ...Matcher) net.Listener {
	mws := matchersToMatchWriters(matchers)
	return m.MatchWithWriters(mws...)
}

func (m *cMux) MatchWithWriters(matchers ...MatchWriter) net.Listener {
	ml := muxListener{
		Listener: m.root,
		//cmux匹配到适合的请求
		connc:    make(chan net.Conn, m.bufLen),
	}
	m.sls = append(m.sls, matchersListener{ss: matchers, l: ml})
	return ml
}

func (l muxListener) Accept() (net.Conn, error) {
	//cmux.server匹配到请求
	c, ok := <-l.connc
	if !ok {
		return nil, ErrListenerClosed
	}
	return c, nil
}

```

