### 启动

- ./etcdmain/main.go/Main 命令行参数初始化
  - 判断是否有环境变量ETCDCOV_ARGS
  - 判断os.Args[1]是否为gateway,grpc-proxy
  - 无参数默认启动etcd

- ./etcdamin/etcd.go/startEtcdOrProxyV2 启动主流程

  - 解析config

  - ```go
    type config struct {
    	ec           embed.Config
    	cp           configProxy
    	cf           configFlags
    	configFile   string
    	printVersion bool
    	ignored      []string
    }
    这里embed.Config就是etcdserver的配置
    具体的配置项参考https://etcd.io/docs/v3.4.0/op-guide/configuration/	
    ```

  - identifyDataDirOrDie 判断data-dir是member|proxy|empty 通过类型判断 执行相应startEtcd|startProxy

  - ./embed/etcd.go/StartEtcd 主要建立peer tcp connection

  - ```
    生成Etcd对象
    type Etcd struct {
    	Peers   []*peerListener
    	Clients []net.Listener
    	// a map of contexts for the servers that serves client requests.
    	sctxs            map[string]*serveCtx
    	metricsListeners []net.Listener
    
    	Server *etcdserver.EtcdServer
    
    	cfg   Config
    	stopc chan struct{}
    	errc  chan error
    
    	closeOnce sync.Once
    }
    e.Server.Start()
    
    ```

    

