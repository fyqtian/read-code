### 服务端接收请求流程

- ```go
  ./etcd/embed/etcd.go/StartEtcd line 236
  
  func StartEtcd(inCfg *Config) (e *Etcd, err error) {
  ...
  	if err = e.serveClients(); err != nil {
  		return e, err
  	}
  ...
  }
  
  func (e *Etcd) serveClients() (err error) {
   ...
      //判断api version
      //default EnableV2=false
      if e.Config().EnableV2 {
  		if len(e.Config().ExperimentalEnableV2V3) > 0 {
  			srv := v2v3.NewServer(e.cfg.logger, v3client.New(e.Server), e.cfg.ExperimentalEnableV2V3)
  			h = v2http.NewClientHandler(e.GetLogger(), srv, e.Server.Cfg.ReqTimeout())
  		} else {
  			h = v2http.NewClientHandler(e.GetLogger(), e.Server, e.Server.Cfg.ReqTimeout())
  		}
  	} else {
  		mux := http.NewServeMux()
          //注册一些基本路由
          // deprecate in 3.5 PUT /config/local/log 
  	 	// /debug/vars
          // /version
          // /metrics
  		// /health 
          etcdhttp.HandleBasic(mux, e.Server)
  		h = mux
  	}
  
  	gopts := []grpc.ServerOption{}
  	if e.cfg.GRPCKeepAliveMinTime > time.Duration(0) {
  		gopts = append(gopts, grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
  			MinTime:             e.cfg.GRPCKeepAliveMinTime,
  			PermitWithoutStream: false,
  		}))
  	}
  	if e.cfg.GRPCKeepAliveInterval > time.Duration(0) &&
  		e.cfg.GRPCKeepAliveTimeout > time.Duration(0) {
  		gopts = append(gopts, grpc.KeepaliveParams(keepalive.ServerParameters{
  			Time:    e.cfg.GRPCKeepAliveInterval,
  			Timeout: e.cfg.GRPCKeepAliveTimeout,
  		}))
  	}
      // start client servers in each goroutine
  	for _, sctx := range e.sctxs {
  		go func(s *serveCtx) {
  			e.errHandler(s.serve(e.Server, &e.cfg.ClientTLSInfo, h, e.errHandler, gopts...))
  		}(sctx)
  	}
   ...
  }
  // serve accepts incoming connections on the listener l,
  // creating a new service goroutine for each. The service goroutines
  // read requests and then call handler to reply to them.
  func (sctx *serveCtx) serve(
     s *etcdserver.EtcdServer,
     tlsinfo *transport.TLSInfo,
     handler http.Handler,
     errHandler func(error),
     gopts ...grpc.ServerOption) (err error){
      
  ...
      //https://github.com/soheilhy/cmux
      m := cmux.New(sctx.l)
      //
  	v3c := v3client.New(s)
  	servElection := v3election.NewElectionServer(v3c)
  	servLock := v3lock.NewLockServer(v3c)
  
  	var gs *grpc.Server
  	defer func() {
  		if err != nil && gs != nil {
  			gs.Stop()
  		}
  	}()
  ...
  if sctx.insecure {
  		gs = v3rpc.Server(s, nil, gopts...)
  		v3electionpb.RegisterElectionServer(gs, servElection)
  		v3lockpb.RegisterLockServer(gs, servLock)
  		if sctx.serviceRegister != nil {
  			sctx.serviceRegister(gs)
  		}
      	//过滤出http2的流 PRI * HTTP/2.0
  		grpcl := m.Match(cmux.HTTP2())
  		go func() { errHandler(gs.Serve(grpcl)) }()
  
  		var gwmux *gw.ServeMux
      	//注册grpc-gateway
  		if s.Cfg.EnableGRPCGateway {
  			gwmux, err = sctx.registerGateway([]grpc.DialOption{grpc.WithInsecure()})
  			if err != nil {
  				return err
  			}
  		}
  
  		httpmux := sctx.createMux(gwmux, handler)
  
  		srvhttp := &http.Server{
  			Handler:  createAccessController(sctx.lg, s, httpmux),
  			ErrorLog: logger, // do not log user error
  		}
      	//过滤出http1的流
  		httpl := m.Match(cmux.HTTP1())
  		go func() { errHandler(srvhttp.Serve(httpl)) }()
  
  		sctx.serversC <- &servers{grpc: gs, http: srvhttp}
  		if sctx.lg != nil {
  			sctx.lg.Info(
  				"serving client traffic insecurely; this is strongly discouraged!",
  				zap.String("address", sctx.l.Addr().String()),
  			)
  		} else {
  			plog.Noticef("serving insecure client requests on %s, this is strongly discouraged!", sctx.l.Addr().String())
  		}
  	}    
      
  ...   
      return m.Serve()
  } 
  
  //
  func (m *cMux) Serve() error {
  	var wg sync.WaitGroup
  
  	defer func() {
  		close(m.donec)
  		wg.Wait()
  
  		for _, sl := range m.sls {
  			close(sl.l.connc)
  			// Drain the connections enqueued for the listener.
  			for c := range sl.l.connc {
  				_ = c.Close()
  			}
  		}
  	}()
  
  	for {
  		c, err := m.root.Accept()
  		if err != nil {
  			if !m.handleErr(err) {
  				return err
  			}
  			continue
  		}
  
  		wg.Add(1)
  		go m.serve(c, m.donec, &wg)
  	}
  }
  
  ```

  