func main() {
   ns, err := net.LookupHost("www.baidu.com")
   if err != nil {
      fmt.Fprintf(os.Stderr, "Err: %s", err.Error())
      return
   }
   for _, n := range ns {
      fmt.Fprintf(os.Stdout, "--%s\n", n)
   }
}

type Resolver struct {
	// PreferGo controls whether Go's built-in DNS resolver is preferred
	// on platforms where it's available. It is equivalent to setting
	// GODEBUG=netdns=go, but scoped to just this resolver.
	PreferGo bool

	// StrictErrors controls the behavior of temporary errors
	// (including timeout, socket errors, and SERVFAIL) when using
	// Go's built-in resolver. For a query composed of multiple
	// sub-queries (such as an A+AAAA address lookup, or walking the
	// DNS search list), this option causes such errors to abort the
	// whole query instead of returning a partial result. This is
	// not enabled by default because it may affect compatibility
	// with resolvers that process AAAA queries incorrectly.
	StrictErrors bool
	
	// Dial optionally specifies an alternate dialer for use by
	// Go's built-in DNS resolver to make TCP and UDP connections
	// to DNS services. The host in the address parameter will
	// always be a literal IP address and not a host name, and the
	// port in the address parameter will be a literal port number
	// and not a service name.
	// If the Conn returned is also a PacketConn, sent and received DNS
	// messages must adhere to RFC 1035 section 4.2.1, "UDP usage".
	// Otherwise, DNS messages transmitted over Conn must adhere
	// to RFC 7766 section 5, "Transport Protocol Selection".
	// If nil, the default dialer is used.
	Dial func(ctx context.Context, network, address string) (Conn, error)
	
	// lookupGroup merges LookupIPAddr calls together for lookups for the same
	// host. The lookupGroup key is the LookupIPAddr.host argument.
	// The return values are ([]IPAddr, error).
	lookupGroup singleflight.Group
	
	// TODO(bradfitz): optional interface impl override hook
	// TODO(bradfitz): Timeout time.Duration?
}


// LookupHost寻找给定的host通过本地的resolver，返回一个切片包含域名的地址
func LookupHost(host string) (addrs []string, err error) {
	return DefaultResolver.LookupHost(context.Background(), host)
}


func (r *Resolver) LookupHost(ctx context.Context, host string) (addrs []string, err error) {
	if host == "" {
		return nil, &DNSError{Err: errNoSuchHost.Error(), Name: host, IsNotFound: true}
	}
	if ip, _ := parseIPZone(host); ip != nil {
		return []string{host}, nil
	}
	return r.lookupHost(ctx, host)
}
// 把s当作IP地址来解析
func parseIPZone(s string) (IP, string) {
	for i := 0; i < len(s); i++ {
		switch s[i] {
		case '.':
			return parseIPv4(s), ""
		case ':':
			return parseIPv6Zone(s)
		}
	}
	return nil, ""
}

func (r *Resolver) lookupHost(ctx context.Context, name string) ([]string, error) {
	ips, err := r.lookupIP(ctx, "ip", name)
	if err != nil {
		return nil, err
	}
	addrs := make([]string, 0, len(ips))
	for _, ip := range ips {
		addrs = append(addrs, ip.String())
	}
	return addrs, nil
}
// windows
func (r *Resolver) lookupIP(ctx context.Context, network, name string) ([]IPAddr, error) {
	// TODO(bradfitz,brainman): use ctx more. See TODO below.

	var family int32 = syscall.AF_UNSPEC
	switch ipVersion(network) {
	case '4':
		family = syscall.AF_INET
	case '6':
		family = syscall.AF_INET6
	}
	
	getaddr := func() ([]IPAddr, error) {
	    // 限制并发数
		acquireThread()
		defer releaseThread()
		hints := syscall.AddrinfoW{
			Family:   family,
			Socktype: syscall.SOCK_STREAM,
			Protocol: syscall.IPPROTO_IP,
		}
		var result *syscall.AddrinfoW
		name16p, err := syscall.UTF16PtrFromString(name)
		if err != nil {
			return nil, &DNSError{Name: name, Err: err.Error()}
		}
	    // 系统调用不明白	
		e := syscall.GetAddrInfoW(name16p, nil, &hints, &result)
		if e != nil {
			err := winError("getaddrinfow", e)
			dnsError := &DNSError{Err: err.Error(), Name: name}
			if err == errNoSuchHost {
				dnsError.IsNotFound = true
			}
			return nil, dnsError
		}
	    // 系统调用不明白
		defer syscall.FreeAddrInfoW(result)
		addrs := make([]IPAddr, 0, 5)
		for ; result != nil; result = result.Next {
			addr := unsafe.Pointer(result.Addr)
			switch result.Family {
			case syscall.AF_INET:
				a := (*syscall.RawSockaddrInet4)(addr).Addr
				addrs = append(addrs, IPAddr{IP: IPv4(a[0], a[1], a[2], a[3])})
			case syscall.AF_INET6:
				a := (*syscall.RawSockaddrInet6)(addr).Addr
				zone := zoneCache.name(int((*syscall.RawSockaddrInet6)(addr).Scope_id))
				addrs = append(addrs, IPAddr{IP: IP{a[0], a[1], a[2], a[3], a[4], a[5], a[6], a[7], a[8], a[9], a[10], a[11], a[12], a[13], a[14], a[15]}, Zone: zone})
			default:
				return nil, &DNSError{Err: syscall.EWINDOWS.Error(), Name: name}
			}
		}
		return addrs, nil
	}
	
	type ret struct {
		addrs []IPAddr
		err   error
	}
	
	var ch chan ret
	// ctx没取消或超时
	if ctx.Err() == nil {
		ch = make(chan ret, 1)
		go func() {
	        // 处理实际
			addr, err := getaddr()
			ch <- ret{addrs: addr, err: err}
		}()
	}
	
	select {
	case r := <-ch:
		return r.addrs, r.err
	case <-ctx.Done():
		// TODO(bradfitz,brainman): cancel the ongoing
		// GetAddrInfoW? It would require conditionally using
		// GetAddrInfoEx with lpOverlapped, which requires
		// Windows 8 or newer. I guess we'll need oldLookupIP,
		// newLookupIP, and newerLookUP.
		//
		// For now we just let it finish and write to the
		// buffered channel.
		return nil, &DNSError{
			Name:      name,
			Err:       ctx.Err().Error(),
			IsTimeout: ctx.Err() == context.DeadlineExceeded,
		}
	}
}