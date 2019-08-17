### 服务发现
在了解 grpc 服务发现之前，我们先来了解一下服务发现的路由方式。一般来说，我们有客户端路由和代理层路由两种方式。

#### 客户端路由
客户端路由模式，也就是调用方负责获取被调用方的地址信息，并使用相应的负载均衡算法发起请求。调用方访问服务注册服务，获取对应的服务 IP 地址和端口，可能还包括对应的服务负载信息（负载均衡算法、服务实例权重等）。调用方通过负载均衡算法选取其中一个发起请求。如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817131529906.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpdWJyb3RoZXI=,size_16,color_FFFFFF,t_70)
server 启动的时候向 config server 注册自身的服务地址，server 正常退出的时候调用接口移除自身地址，通过定时心跳保证服务是否正常以及地址的有效性。

#### 代理层路由
代理层路由，不是由调用方去获取被调方的地址，而是通过代理的方式，由代理去获取被调方的地址、发起调用请求。如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817132105444.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpdWJyb3RoZXI=,size_16,color_FFFFFF,t_70)
代理层路由这种模式，对 server 的寻址不再是由 client 去实现，而是由代理去实现。client 只是会对代理层发起简单请求，代理层去进行 server 寻址、负载均衡等。

grpc 官方介绍的服务发现流程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817134928571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpdWJyb3RoZXI=,size_16,color_FFFFFF,t_70)
由这张图可以看出，grpc 是使用客户端路由的方式。具体的过程我们在介绍负载均衡时再继续介绍


### grpc 服务发现
之前在介绍 grpc client 时谈到了 resolver 这个类，对下面这段代码并未做详细介绍，我们来详细看看

	if cc.dopts.resolverBuilder == nil {
			// Only try to parse target when resolver builder is not already set.
			cc.parsedTarget = parseTarget(cc.target)
			grpclog.Infof("parsed scheme: %q", cc.parsedTarget.Scheme)
			cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
			if cc.dopts.resolverBuilder == nil {
				// If resolver builder is still nil, the parsed target's scheme is
				// not registered. Fallback to default resolver and set Endpoint to
				// the original target.
				grpclog.Infof("scheme %q not registered, fallback to default scheme", cc.parsedTarget.Scheme)
				cc.parsedTarget = resolver.Target{
					Scheme:   resolver.GetDefaultScheme(),
					Endpoint: target,
				}
				cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
			}
		} else {
			cc.parsedTarget = resolver.Target{Endpoint: target}
		}

这段代码主要干了两件事情，parseTarget 和 resolver.Get 获取了一个 resolverBuilder

parseTarget 其实就是将 target 赋值给了 resolver  target 对象的 endpoint 属性，如下

	func parseTarget(target string) (ret resolver.Target) {
		var ok bool
		ret.Scheme, ret.Endpoint, ok = split2(target, "://")
		if !ok {
			return resolver.Target{Endpoint: target}
		}
		ret.Authority, ret.Endpoint, ok = split2(ret.Endpoint, "/")
		if !ok {
			return resolver.Target{Endpoint: target}
		}
		return ret
	}

这里来看 resolver.Get 方法 ，这里从一个 map 中取出了一个 Builder
	
	var (
		// m is a map from scheme to resolver builder.
		m = make(map[string]Builder)
		// defaultScheme is the default scheme to use.
		defaultScheme = "passthrough"
	)

	func Get(scheme string) Builder {
		if b, ok := m[scheme]; ok {
			return b
		}
		return nil
	}

Builder 是在 resolver 中定义的，在了解 Builder 是啥之前，我们先来看看 resolver 这个结构体

### resolver

	// Package resolver defines APIs for name resolution in gRPC.
	// All APIs in this package are experimental.

resolver 主要提供了一个名字解析的规范，所有的名字解析服务可以实现这个规范，包括 dns 解析类 dns_resolver 就是实现了这个规范的一个解析器。

resolver 中定义了 Builder ，通过调用 Build 去获取一个 resolver 实例

	// Builder creates a resolver that will be used to watch name resolution updates.
	type Builder interface {
		// Build creates a new resolver for the given target.
		//
		// gRPC dial calls Build synchronously, and fails if the returned error is
		// not nil.
		Build(target Target, cc ClientConn, opts BuildOption) (Resolver, error)
		// Scheme returns the scheme supported by this resolver.
		// Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.
		Scheme() string
	}

我们在调用 Dial 方法发起 rpc 请求之前需要创建一个 ClientConn 连接，在 DialContext 这个方法中对 ClientConn 各属性进行了赋值，其中有一行代码就完成了 build resolver 的工作。

	// Build the resolver.
	rWrapper, err := newCCResolverWrapper(cc)

	func newCCResolverWrapper(cc *ClientConn) (*ccResolverWrapper, error) {
		rb := cc.dopts.resolverBuilder
		if rb == nil {
			return nil, fmt.Errorf("could not get resolver for scheme: %q", cc.parsedTarget.Scheme)
		}
	
		ccr := &ccResolverWrapper{
			cc:     cc,
			addrCh: make(chan []resolver.Address, 1),
			scCh:   make(chan string, 1),
		}
	
		var err error
		ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, resolver.BuildOption{DisableServiceConfig: cc.dopts.disableServiceConfig})
		if err != nil {
			return nil, err
		}
		return ccr, nil
	}

不出意料，我们之前通过 get 去获取了一个 Builder， 这里调用了 Builder 的 Build 方法产生一个 resolver。

### register() 

上面我们说到了，resolver 通过 get 方法，根据一个 string key 去一个 builder map 中获取一个 builder，这个 map 在 resolver 中初始化如下，那么是怎么进行赋值的呢？

	var (
		// m is a map from scheme to resolver builder.
		m = make(map[string]Builder)
		// defaultScheme is the default scheme to use.
		defaultScheme = "passthrough"
	)

我们猜测肯定会有一个服务注册的过程，果然看到了一个 Register 方法

	func Register(b Builder) {
		m[b.Scheme()] = b
	}

所有的 resolver 实现类通过 Register 方法去实现 Builder 的注册，比如 grpc 提供的 dnsResolver 这个类中调用了 init 方法，在服务初始化时实现了 Builder 的注册

	func init() {
		resolver.Register(NewBuilder())
	}

### 获取服务地址
resolver 和 builder 都是 interface，也就是说它们只是定义了一套规则。具体实现由实现他们的子类去完成。例如在 helloworld 例子中，默认是通过默认的 passthrough 这个 scheme 去获取的 passthroughResolver 和 passthroughBuilder，我们来看 passthroughBuilder  的 Build 方法返回了一个带有 address 的 resolver，这个地址就是 server 的地址列表。在 helloworld demo 中，就是 “localhost:50051”。

	func (*passthroughBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
		r := &passthroughResolver{
			target: target,
			cc:     cc,
		}
		r.start()
		return r, nil
	}

	func (r *passthroughResolver) start() {
		r.cc.UpdateState(resolver.State{Addresses: []resolver.Address{{Addr: r.target.Endpoint}}})
	}



### dns_resolver
grpc 支持自定义 resolver 实现服务发现。同时 grpc 官方提供了一个基于 dns 的服务发现 resolver，这就是 dns_resolver，dns_resolver 通过 Build() 创建一个 resolver 实例，我们来具体看一下 Build() 方法：

	// Build creates and starts a DNS resolver that watches the name resolution of the target.
	func (b *dnsBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
		host, port, err := parseTarget(target.Endpoint, defaultPort)
		if err != nil {
			return nil, err
		}
	
		// IP address.
		if net.ParseIP(host) != nil {
			host, _ = formatIP(host)
			addr := []resolver.Address{{Addr: host + ":" + port}}
			i := &ipResolver{
				cc: cc,
				ip: addr,
				rn: make(chan struct{}, 1),
				q:  make(chan struct{}),
			}
			cc.NewAddress(addr)
			go i.watcher()
			return i, nil
		}
	
		// DNS address (non-IP).
		ctx, cancel := context.WithCancel(context.Background())
		d := &dnsResolver{
			freq:                 b.minFreq,
			backoff:              backoff.Exponential{MaxDelay: b.minFreq},
			host:                 host,
			port:                 port,
			ctx:                  ctx,
			cancel:               cancel,
			cc:                   cc,
			t:                    time.NewTimer(0),
			rn:                   make(chan struct{}, 1),
			disableServiceConfig: opts.DisableServiceConfig,
		}
	
		if target.Authority == "" {
			d.resolver = defaultResolver
		} else {
			d.resolver, err = customAuthorityResolver(target.Authority)
			if err != nil {
				return nil, err
			}
		}
	
		d.wg.Add(1)
		go d.watcher()
		return d, nil
	}

在 Build 方法中，我们没有看到对 server address 寻址的过程，仔细找找，发现了一个 watcher 方法，如下：

		go d.watcher()

看一下 watcher 方法，发现它其实是一个监控进程，顾名思义作用是监控我们产生的 resolver 的状态，这里使用了一个 for 循环无限监听，通过 chan 进行消息通知。

		func (d *dnsResolver) watcher() {
			defer d.wg.Done()
			for {
				select {
				case <-d.ctx.Done():
					return
				case <-d.t.C:
				case <-d.rn:
					if !d.t.Stop() {
						// Before resetting a timer, it should be stopped to prevent racing with
						// reads on it's channel.
						<-d.t.C
					}
				}
		
				result, sc := d.lookup()
				// Next lookup should happen within an interval defined by d.freq. It may be
				// more often due to exponential retry on empty address list.
				if len(result) == 0 {
					d.retryCount++
					d.t.Reset(d.backoff.Backoff(d.retryCount))
				} else {
					d.retryCount = 0
					d.t.Reset(d.freq)
				}
				d.cc.NewServiceConfig(sc)
				d.cc.NewAddress(result)
		
				// Sleep to prevent excessive re-resolutions. Incoming resolution requests
				// will be queued in d.rn.
				t := time.NewTimer(minDNSResRate)
				select {
				case <-t.C:
				case <-d.ctx.Done():
					t.Stop()
					return
				}
			}
		}

我们定位到里面的 lookup 方法。

				result, sc := d.lookup()

进入 lookup 方法，发现它调用了 lookupSRV 这个方法

	func (d *dnsResolver) lookup() ([]resolver.Address, string) {
		newAddrs := d.lookupSRV()
		// Support fallback to non-balancer address.
		newAddrs = append(newAddrs, d.lookupHost()...)
		if d.disableServiceConfig {
			return newAddrs, ""
		}
		sc := d.lookupTXT()
		return newAddrs, canaryingSC(sc)
	}

继续追踪，lookupSRV 这个方法最终其实调用了 go 源码包 net 包下的 的 lookupSRV 方法，这个方法实现了 dns 协议对指定的service服务，protocol协议以及name域名进行srv查询，返回server 的 address 列表。经过层层解剖，我们终于找到了返回 server 的 address list 的代码。

		_, srvs, err := d.resolver.LookupSRV(d.ctx, "grpclb", "tcp", d.host)

		...
	
		func (r *Resolver) LookupSRV(ctx context.Context, service, proto, name string) (cname string, addrs []*SRV, err error) {
			return r.lookupSRV(ctx, service, proto, name)
		}

### 总结

总结一下， grpc 的服务发现，主要通过 resolver 接口去定义，支持业务自己实现服务发现的 resolver。 grpc 提供了默认的 passthrough_resolver，不进行地址解析，直接将 client 发起请求时指定的 address （例如 helloworld client 指定地址为 “localhost:50051” ）当成 server address。同时，假如业务使用 dns 进行服务发现，grpc 提供了 dns_resolver，通过对指定的service服务，protocol协议以及name域名进行srv查询，来返回 server 的 address 列表。


