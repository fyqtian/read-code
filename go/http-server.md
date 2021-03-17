### http-server

https://draveness.me/golang/docs/part4-advanced/ch09-stdlib/golang-net-http/

Go 语言的 [`net/http`](https://golang.org/pkg/net/http/) 中同时包好了 HTTP 客户端和服务端的实现，为了支持更好的扩展性，它引入了 [`net/http.RoundTripper`](https://draveness.me/golang/tree/net/http.RoundTripper) 和 [`net/http.Handler`](https://draveness.me/golang/tree/net/http.Handler) 两个接口。[`net/http.RoundTripper`](https://draveness.me/golang/tree/net/http.RoundTripper) 是用来表示执行 HTTP 请求的接口，调用方将请求作为参数可以获取请求对应的响应，而 [`net/http.Handler`](https://draveness.me/golang/tree/net/http.Handler) 主要用于 HTTP 服务器响应客户端的请求：



```go
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```



HTTP 请求的接收方可以实现 [`net/http.Handler`](https://draveness.me/golang/tree/net/http.Handler) 接口，其中实现了处理 HTTP 请求的逻辑，处理的过程中会调用 [`net/http.ResponseWriter`](https://draveness.me/golang/tree/net/http.ResponseWriter) 接口的方法构造 HTTP 响应，它提供的三个接口 `Header`、`Write` 和 `WriteHeader` 分别会获取 HTTP 响应、将数据写入负载以及写入响应头：



```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

type ResponseWriter interface {
	Header() Header
	Write([]byte) (int, error)
	WriteHeader(statusCode int)
}
```



客户端和服务端面对的都是双向的 HTTP 请求与响应，客户端构建请求并等待响应，服务端处理请求并返回响应。HTTP 请求和响应在标准库中不止有一种实现，它们都包含了层级结构，标准库中的 [`net/http.RoundTripper`](https://draveness.me/golang/tree/net/http.RoundTripper) 包含如下所示的层级结构

<img src="..\images\2020-05-18-15897352888419-golang-roundtripper.png" alt="2020-05-18-15897352888419-golang-roundtripper" style="zoom:50%;" />



```go
func main() {
   http.ListenAndServe(":9999", nil)
    
    要自定义字段属性 直接使用 http.Server{}.ListenAndServe
}



// A Server defines parameters for running an HTTP server.
// The zero value for Server is a valid configuration.
type Server struct {
	// Addr optionally specifies the TCP address for the server to listen on,
	// in the form "host:port". If empty, ":http" (port 80) is used.
	// The service names are defined in RFC 6335 and assigned by IANA.
	// See net.Dial for details of the address format.
	Addr string

	Handler Handler // handler to invoke, http.DefaultServeMux if nil

	// TLSConfig optionally provides a TLS configuration for use
	// by ServeTLS and ListenAndServeTLS. Note that this value is
	// cloned by ServeTLS and ListenAndServeTLS, so it's not
	// possible to modify the configuration with methods like
	// tls.Config.SetSessionTicketKeys. To use
	// SetSessionTicketKeys, use Server.Serve with a TLS Listener
	// instead.
	TLSConfig *tls.Config

	// ReadTimeout is the maximum duration for reading the entire
	// request, including the body.
	//
	// Because ReadTimeout does not let Handlers make per-request
	// decisions on each request body's acceptable deadline or
	// upload rate, most users will prefer to use
	// ReadHeaderTimeout. It is valid to use them both.
	ReadTimeout time.Duration

	// ReadHeaderTimeout is the amount of time allowed to read
	// request headers. The connection's read deadline is reset
	// after reading the headers and the Handler can decide what
	// is considered too slow for the body. If ReadHeaderTimeout
	// is zero, the value of ReadTimeout is used. If both are
	// zero, there is no timeout.
	ReadHeaderTimeout time.Duration

	// WriteTimeout is the maximum duration before timing out
	// writes of the response. It is reset whenever a new
	// request's header is read. Like ReadTimeout, it does not
	// let Handlers make decisions on a per-request basis.
	WriteTimeout time.Duration

	// IdleTimeout is the maximum amount of time to wait for the
	// next request when keep-alives are enabled. If IdleTimeout
	// is zero, the value of ReadTimeout is used. If both are
	// zero, there is no timeout.
	IdleTimeout time.Duration

	// MaxHeaderBytes controls the maximum number of bytes the
	// server will read parsing the request header's keys and
	// values, including the request line. It does not limit the
	// size of the request body.
	// If zero, DefaultMaxHeaderBytes is used.
	MaxHeaderBytes int

	// TLSNextProto optionally specifies a function to take over
	// ownership of the provided TLS connection when an ALPN
	// protocol upgrade has occurred. The map key is the protocol
	// name negotiated. The Handler argument should be used to
	// handle HTTP requests and will initialize the Request's TLS
	// and RemoteAddr if not already set. The connection is
	// automatically closed when the function returns.
	// If TLSNextProto is not nil, HTTP/2 support is not enabled
	// automatically.
	TLSNextProto map[string]func(*Server, *tls.Conn, Handler)

	// ConnState specifies an optional callback function that is
	// called when a client connection changes state. See the
	// ConnState type and associated constants for details.
	ConnState func(net.Conn, ConnState)

	// ErrorLog specifies an optional logger for errors accepting
	// connections, unexpected behavior from handlers, and
	// underlying FileSystem errors.
	// If nil, logging is done via the log package's standard logger.
	ErrorLog *log.Logger

	// BaseContext optionally specifies a function that returns
	// the base context for incoming requests on this server.
	// The provided Listener is the specific Listener that's
	// about to start accepting requests.
	// If BaseContext is nil, the default is context.Background().
	// If non-nil, it must return a non-nil context.
	BaseContext func(net.Listener) context.Context

	// ConnContext optionally specifies a function that modifies
	// the context used for a new connection c. The provided ctx
	// is derived from the base context and has a ServerContextKey
	// value.
	ConnContext func(ctx context.Context, c net.Conn) context.Context

	inShutdown atomicBool // true when when server is in shutdown

	disableKeepAlives int32     // accessed atomically.
	nextProtoOnce     sync.Once // guards setupHTTP2_* init
	nextProtoErr      error     // result of http2.ConfigureServer if used

	mu         sync.Mutex
	listeners  map[*net.Listener]struct{}
	activeConn map[*conn]struct{}
	doneChan   chan struct{}
	onShutdown []func()
}


// ListenAndServe监听再tcp的地址，然后调用处理程序处理传入连接上的请求。
// 连接得tcp保持keepalive
// handler通常是nil，DefaultServeMux默认被使用
// ListenAndServe通常返回一个非nil得错误
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}


// 如果 srv.Addr是空得,:http被使用
// ListenAndServe通常返回一个非nil得错误，再shutdown或者close，返回ErrServerClosed
func (srv *Server) ListenAndServe() error {
    // 如果已经标记关闭
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
    //建立tcp
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}


// Serve接收再Listener得连接请求，创建一个新的服务goroutine为每个请求，服务端goroutine读取请求并调用srv.Handler
// HTTP/2 支持只有在启用tls连接，并且配置h2 TLS Config.NextProtos
// Serve通常返回一个非空得error并关闭tcp 连接

func (srv *Server) Serve(l net.Listener) error {
	
	origListener := l
    //包装了sync.once
	l = &onceCloseListener{Listener: l}
	defer l.Close()
	//设置启用h2
	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}
	// 加入到srv.listeners map中
	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
	defer srv.trackListener(&l, false)
	//给上层提供了回调方法 传参Listener返回context
	baseCtx := context.Background()
	if srv.BaseContext != nil {
		baseCtx = srv.BaseContext(origListener)
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}
	// 重试回退
	var tempDelay time.Duration // how long to sleep on accept failure
	// context包含了当前server
    // 通过http.Request.Context()
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
        //客户端连接
		rw, err := l.Accept()
		if err != nil {
			select {
                //判断是否关闭
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
            //判断是否是临时错误，休眠等待重试
			if ne, ok := err.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return err
		}
        //注册得ConnContext，可以拿到当前得tcp连接
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
        //连接成功 清空回退
		tempDelay = 0
        //包装一个连接
		c := srv.newConn(rw)
        //标记一个状态 放到activeConn中
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve(connCtx)
	}
}



// Serve a new connection.
func (c *conn) serve(ctx context.Context) {
    // 客户端地址
	c.remoteAddr = c.rwc.RemoteAddr().String()
    // ctx存入本地地址
	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
    // 注册recover
	defer func() {
		if err := recover(); err != nil && err != ErrAbortHandler {
			const size = 64 << 10
			buf := make([]byte, size)
			buf = buf[:runtime.Stack(buf, false)]
			c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
		}
		if !c.hijacked() {
			c.close()
			c.setState(c.rwc, StateClosed)
		}
	}()
	//如果是tls连接 没仔细看详细的
	if tlsConn, ok := c.rwc.(*tls.Conn); ok {
        //设置读取超时
		if d := c.server.ReadTimeout; d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
		}
        //设置写入超时
		if d := c.server.WriteTimeout; d != 0 {
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}
        // tls握手
		if err := tlsConn.Handshake(); err != nil {
			// If the handshake failed due to the client not speaking
			// TLS, assume they're speaking plaintext HTTP and write a
			// 400 response on the TLS conn's underlying net.Conn.
            // 如果握手失败是因为client没有使用tls，假设他们在用http文本协议 写入一个400响应
            // tlsRecordHeaderLooksLikeHTT 读取http 第一行
			if re, ok := err.(tls.RecordHeaderError); ok && re.Conn != nil && tlsRecordHeaderLooksLikeHTTP(re.RecordHeader) {
				io.WriteString(re.Conn, "HTTP/1.0 400 Bad Request\r\n\r\nClient sent an HTTP request to an HTTPS server.\n")
				re.Conn.Close()
				return
			}
			c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
			return
		}
		c.tlsState = new(tls.ConnectionState)
		*c.tlsState = tlsConn.ConnectionState()
		if proto := c.tlsState.NegotiatedProtocol; validNextProto(proto) {
			if fn := c.server.TLSNextProto[proto]; fn != nil {
				h := initALPNRequest{ctx, tlsConn, serverHandler{c.server}}
				fn(c.server, tlsConn, h)
			}
			return
		}
	}

	// HTTP/1.x from here on.

	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()
	// 设置各类读写缓冲
	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
        //解析http 请求头
        // 判断header是否合法
        // 判断是否要upgrade协议
		w, err := c.readRequest(ctx)
		if c.r.remain != c.server.initialReadLimitSize() {
			// If we read any bytes off the wire, we're active.
			c.setState(c.rwc, StateActive)
		}
        // 如果err不为nil说明解析头失败 直接返回
		if err != nil {
			const errorHeaders = "\r\nContent-Type: text/plain; charset=utf-8\r\nConnection: close\r\n\r\n"

			switch {
			case err == errTooLarge:
				// Their HTTP client may or may not be
				// able to read this if we're
				// responding to them and hanging up
				// while they're still writing their
				// request. Undefined behavior.
				const publicErr = "431 Request Header Fields Too Large"
				fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
				c.closeWriteAndWait()
				return

			case isUnsupportedTEError(err):
				// Respond as per RFC 7230 Section 3.3.1 which says,
				//      A server that receives a request message with a
				//      transfer coding it does not understand SHOULD
				//      respond with 501 (Unimplemented).
				code := StatusNotImplemented

				// We purposefully aren't echoing back the transfer-encoding's value,
				// so as to mitigate the risk of cross side scripting by an attacker.
				fmt.Fprintf(c.rwc, "HTTP/1.1 %d %s%sUnsupported transfer encoding", code, StatusText(code), errorHeaders)
				return

			case isCommonNetReadError(err):
				return // don't reply

			default:
				publicErr := "400 Bad Request"
				if v, ok := err.(badRequestError); ok {
					publicErr = publicErr + ": " + string(v)
				}

				fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
				return
			}
		}

		// Expect 100 Continue support
		req := w.req
		if req.expectsContinue() {
			if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
				// Wrap the Body reader with one that replies on the connection
				req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
				w.canWriteContinue.setTrue()
			}
		} else if req.Header.get("Expect") != "" {
			w.sendExpectationFailed()
			return
		}

		c.curReq.Store(w)

		if requestBodyRemains(req.Body) {
			registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
		} else {
			w.conn.r.startBackgroundRead()
		}

		// HTTP cannot have multiple simultaneous active requests.[*]
		// Until the server replies to this request, it can't read another,
		// so we might as well run the handler in this goroutine.
		// [*] Not strictly true: HTTP pipelining. We could let them all process
		// in parallel even if their responses need to be serialized.
		// But we're not going to implement HTTP pipelining because it
		// was never deployed in the wild and the answer is HTTP/2.
        
        // 进到实际路由 DefaultServeMux
		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.cancelCtx()
       	// 如果hijacked 请求结束
		if c.hijacked() {
			return
		}
        //清理状态
		w.finishRequest()
        // 判断tcp是否能复用
       //
		if !w.shouldReuseConnection() {
			if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
				c.closeWriteAndWait()
			}
			return
		}
        // 保存状态
		c.setState(c.rwc, StateIdle)
		c.curReq.Store((*response)(nil))
		//如果服务器 未开启长连接
		if !w.conn.server.doKeepAlives() {
			// We're in shutdown mode. We might've replied
			// to the user without "Connection: close" and
			// they might think they can send another
			// request, but such is life with HTTP/1.1.
			return
		}
		// 计算连接空闲时间
		if d := c.server.idleTimeout(); d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
			if _, err := c.bufr.Peek(4); err != nil {
				return
			}
		}
        //设置 读取超时 进入下次循环等待读取
		c.rwc.SetReadDeadline(time.Time{})
	}
}

```



默认路由分发

```go

// ServeMux is an HTTP request multiplexer.
// It matches the URL of each incoming request against a list of registered
// patterns and calls the handler for the pattern that
// most closely matches the URL.
//
// Patterns name fixed, rooted paths, like "/favicon.ico",
// or rooted subtrees, like "/images/" (note the trailing slash).
// Longer patterns take precedence over shorter ones, so that
// if there are handlers registered for both "/images/"
// and "/images/thumbnails/", the latter handler will be
// called for paths beginning "/images/thumbnails/" and the
// former will receive requests for any other paths in the
// "/images/" subtree.
//
// Note that since a pattern ending in a slash names a rooted subtree,
// the pattern "/" matches all paths not matched by other registered
// patterns, not just the URL with Path == "/".
//
// If a subtree has been registered and a request is received naming the
// subtree root without its trailing slash, ServeMux redirects that
// request to the subtree root (adding the trailing slash). This behavior can
// be overridden with a separate registration for the path without
// the trailing slash. For example, registering "/images/" causes ServeMux
// to redirect a request for "/images" to "/images/", unless "/images" has
// been registered separately.
//
// Patterns may optionally begin with a host name, restricting matches to
// URLs on that host only. Host-specific patterns take precedence over
// general patterns, so that a handler might register for the two patterns
// "/codesearch" and "codesearch.google.com/" without also taking over
// requests for "http://www.google.com/".
//
// ServeMux also takes care of sanitizing the URL request path and the Host
// header, stripping the port number and redirecting any request containing . or
// .. elements or repeated slashes to an equivalent, cleaner URL.
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

相当于对对实际的handler包了一层
// ServeHTTP dispatches the request to the handler whose
// pattern most closely matches the request URL.
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
   if r.RequestURI == "*" {
      if r.ProtoAtLeast(1, 1) {
         w.Header().Set("Connection", "close")
      }
      w.WriteHeader(StatusBadRequest)
      return
   }
   h, _ := mux.Handler(r)
   h.ServeHTTP(w, r)
}
```