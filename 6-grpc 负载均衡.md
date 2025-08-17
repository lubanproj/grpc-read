> 本文原发表于：https://diu.life/lessons/grpc-read/grpc-load-balancing/
> 最新版本请访问原文链接

### grpc负载均衡
#### 负载均衡流程
grpc 官方的 doc 中介绍了，grpc 的负载均衡是基于一次请求而不是一次连接的。也就是说，假如所有的请求都来自同一个客户端的连接，这些请求还是会被均衡到所有服务器。

整个 grpc 负载均衡流程如下图：

![ grpc 负载均衡](https://img-blog.csdnimg.cn/20190818144901158.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpdWJyb3RoZXI=,size_16,color_FFFFFF,t_70)
1、启动时，grpc client 通过服名字解析服务得到一个 address list，每个 address 将指示它是服务器地址还是负载平衡器地址，以及指示要哪个客户端负载平衡策略的服务配置（例如，round_robin 或 grpclb）

2、客户端实例化负载均衡策略
如果解析程序返回的任何一个地址是负载均衡器地址，则无论 service config 中定义了什么负载均衡策略，客户端都将使用grpclb策略。否则，客户端将使用 service config 中定义的负载均衡策略。如果服务配置未请求负载均衡策略，则客户端将默认使用选择第一个可用服务器地址的策略。

3、负载平衡策略为每个服务器地址创建一个 subchannel，假如是 grpclb 策略，客户端会根据名字解析服务返回的地址列表，请求负载均衡器，由负载均衡器决定请求哪个 subConn，然后打开一个数据流，对这个 subConn 中的所有服务器 adress 都建立连接，从而实现 client stream 的效果

4、当有rpc请求时，负载均衡策略决定哪个子通道即grpc服务器将接收请求，当可用服务器为空时客户端的请求将被阻塞。


#### 源码实现
接下来我们来看一下源码里面关于 grpc 负载均衡的实现，这里主要分为初始化 balancer 和寻址两步

#### 1. 初始化 balancer

之前介绍 grpc 服务发现时，我们知道了通过 dns_resolver 的 lookup 方法可以得到一个 address list，那么拿到地址列表之后具体干了什么事呢？下面我们来看看

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

这里调用了 NewAddress 方法，在 NewAddress 这个方法里面又调用了 updateResolverState 这个方法，对负载均衡器的初始化就是在这个方法中进行的，如下：

		if cc.dopts.balancerBuilder == nil {
			// Only look at balancer types and switch balancer if balancer dial
			// option is not set.
			var newBalancerName string
			if cc.sc != nil && cc.sc.lbConfig != nil {
				newBalancerName = cc.sc.lbConfig.name
				balCfg = cc.sc.lbConfig.cfg
			} else {
				var isGRPCLB bool
				for _, a := range s.Addresses {
					if a.Type == resolver.GRPCLB {
						isGRPCLB = true
						break
					}
				}
				if isGRPCLB {
					newBalancerName = grpclbName
				} else if cc.sc != nil && cc.sc.LB != nil {
					newBalancerName = *cc.sc.LB
				} else {
					newBalancerName = PickFirstBalancerName
				}
			}
			cc.switchBalancer(newBalancerName)
		} else if cc.balancerWrapper == nil {
			// Balancer dial option was set, and this is the first time handling
			// resolved addresses. Build a balancer with dopts.balancerBuilder.
			cc.curBalancerName = cc.dopts.balancerBuilder.Name()
			cc.balancerWrapper = newCCBalancerWrapper(cc, cc.dopts.balancerBuilder, cc.balancerBuildOpts)
		}
	
		cc.balancerWrapper.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})

之前说到了我们在 dns_resolver 中查找 address 时是通过 grpclb 去进行查找的，所以它返回的 resolver 的策略就是 grpclb 策略。这里会进入到 switchBalancer 方法，我们来看看这个方法干了啥

	func (cc *ClientConn) switchBalancer(name string) {

		builder := balancer.Get(name)
		
		...
	
		cc.curBalancerName = builder.Name()
		cc.balancerWrapper = newCCBalancerWrapper(cc, builder, cc.balancerBuildOpts)
	}

这里通过 grpclb 这个 name 去获取到了 grpclb 策略的一个 balancer 实现，然后调用了 newCCBalancerWrapper 这个方法，继续跟踪

	func newCCBalancerWrapper(cc *ClientConn, b balancer.Builder, bopts balancer.BuildOptions) *ccBalancerWrapper {
		ccb := &ccBalancerWrapper{
			cc:               cc,
			stateChangeQueue: newSCStateUpdateBuffer(),
			ccUpdateCh:       make(chan *balancer.ClientConnState, 1),
			done:             make(chan struct{}),
			subConns:         make(map[*acBalancerWrapper]struct{}),
		}
		go ccb.watcher()
		ccb.balancer = b.Build(ccb, bopts)
		return ccb
	}

来看一下这里的 Build 方法，去 grpclb 这个策略的实现类里面看，发现它返回了一个 lbBalancer 实例

	func (b *lbBuilder) Build(cc balancer.ClientConn, opt balancer.BuildOptions) balancer.Balancer {
		...
		lb := &lbBalancer{
			cc:              newLBCacheClientConn(cc),
			target:          opt.Target.Endpoint,
			opt:             opt,
			fallbackTimeout: b.fallbackTimeout,
			doneCh:          make(chan struct{}),
	
			manualResolver: r,
			subConns:       make(map[resolver.Address]balancer.SubConn),
			scStates:       make(map[balancer.SubConn]connectivity.State),
			picker:         &errPicker{err: balancer.ErrNoSubConnAvailable},
			clientStats:    newRPCStats(),
			backoff:        defaultBackoffConfig, // TODO: make backoff configurable.
		}
		...
		return lb
	}

#### 2.  寻址
之前我们说到了，helloworld demo 中 client 发送请求主要分为三步，对 balancer 的初始化其实是在第一步 grpc.Dial 时初始化 dialContext 时完成的。那么寻址过程，就是在第三步调用 sayHello 时完成的。

我们进入 sayHello ——> c.cc.Invoke ——> invoke ——> newClientStream 方法中，有下面一段代码：

	if err := cs.newAttemptLocked(sh, trInfo); err != nil {
			cs.finish(err)
			return nil, err
		}

进入 newAttemptLocked 方法，如下：

	func (cs *clientStream) newAttemptLocked(sh stats.Handler, trInfo *traceInfo) error {
		cs.attempt = &csAttempt{
			cs:           cs,
			dc:           cs.cc.dopts.dc,
			statsHandler: sh,
			trInfo:       trInfo,
		}
	
		if err := cs.ctx.Err(); err != nil {
			return toRPCErr(err)
		}
		t, done, err := cs.cc.getTransport(cs.ctx, cs.callInfo.failFast, cs.callHdr.Method)
		...
	}

我们发现它调用了 getTransport 方法，进入这个方法，我们找到了 pick 方法的调用

		func (cc *ClientConn) getTransport(ctx context.Context, failfast bool, method string) (transport.ClientTransport, func(balancer.DoneInfo), error) {
		t, done, err := cc.blockingpicker.pick(ctx, failfast, balancer.PickOptions{
			FullMethodName: method,
		})
		if err != nil {
			return nil, nil, toRPCErr(err)
		}
		return t, done, nil
	}

pick 方法即是具体寻址的方法，仔细看 pick 方法，它先 Pick 获取了一个 SubConn，SubConn 结构体中包含了一个 address list，然后它会对每一个 address 都会发送 rpc 请求。

	func (bp *pickerWrapper) pick(ctx context.Context, failfast bool, opts balancer.PickOptions) (transport.ClientTransport, func(balancer.DoneInfo), error) {
		var ch chan struct{}
	
		for {
			
			...
			p := bp.picker
			...
			subConn, done, err := p.Pick(ctx, opts)
	
			...	
		}
	}

它调用了 pickWrapper 中的 Pick 方法，在第一步初始化 balancer 的时候我们说到，它返回的其实是 lbBalancer 实例，所以这里去看 lbBalancer 实例的 Pick 实现

	func (p *lbPicker) Pick(ctx context.Context, opts balancer.PickOptions) (balancer.SubConn, func(balancer.DoneInfo), error) {
		p.mu.Lock()
		defer p.mu.Unlock()
	
		// Layer one roundrobin on serverList.
		s := p.serverList[p.serverListNext]
		p.serverListNext = (p.serverListNext + 1) % len(p.serverList)
	
		// If it's a drop, return an error and fail the RPC.
		if s.Drop {
			p.stats.drop(s.LoadBalanceToken)
			return nil, nil, status.Errorf(codes.Unavailable, "request dropped by grpclb")
		}
	
		// If not a drop but there's no ready subConns.
		if len(p.subConns) <= 0 {
			return nil, nil, balancer.ErrNoSubConnAvailable
		}
	
		// Return the next ready subConn in the list, also collect rpc stats.
		sc := p.subConns[p.subConnsNext]
		p.subConnsNext = (p.subConnsNext + 1) % len(p.subConns)
		done := func(info balancer.DoneInfo) {
			if !info.BytesSent {
				p.stats.failedToSend()
			} else if info.BytesReceived {
				p.stats.knownReceived()
			}
		}
		return sc, done, nil
	}

可以看到这其实是一个轮询实现。用一个指针表示这次取的位置，取过之后就更新这个指针为下一位。Pick 的返回是一个 SubConn 结构，SubConn 里面就包含了 server 的地址列表，此时寻址就完成了。

3、发起 request

寻址完成之后，我们得到了包含 server 地址列表的 SubConn，接下来是如何发送请求的呢？在 pick 方法中接着往下看，发现了这段代码。

		acw, ok := subConn.(*acBalancerWrapper)
		if !ok {
			grpclog.Error("subconn returned from pick is not *acBalancerWrapper")
			continue
		}
		if t, ok := acw.getAddrConn().getReadyTransport(); ok {
			if channelz.IsOn() {
				return t, doneChannelzWrapper(acw, done), nil
			}
			return t, done, nil
		}

这段代码先将 SubConn 转换成了一个 acBalancerWrapper ，然后获取其中的 addrConn 对象，接着调用 getReadyTransport 方法，如下：

	func (ac *addrConn) getReadyTransport() (transport.ClientTransport, bool) {
		ac.mu.Lock()
		if ac.state == connectivity.Ready && ac.transport != nil {
			t := ac.transport
			ac.mu.Unlock()
			return t, true
		}
		var idle bool
		if ac.state == connectivity.Idle {
			idle = true
		}
		ac.mu.Unlock()
		// Trigger idle ac to connect.
		if idle {
			ac.connect()
		}
		return nil, false
	}

getReadyTransport 这个方法返回一个 Ready 状态的网络连接，假如连接状态是 IDLE 状态，会调用 connect 方法去进行客户端连接，connect 方法如下：

	func (ac *addrConn) connect() error {
		ac.mu.Lock()
		if ac.state == connectivity.Shutdown {
			ac.mu.Unlock()
			return errConnClosing
		}
		if ac.state != connectivity.Idle {
			ac.mu.Unlock()
			return nil
		}
		// Update connectivity state within the lock to prevent subsequent or
		// concurrent calls from resetting the transport more than once.
		ac.updateConnectivityState(connectivity.Connecting)
		ac.mu.Unlock()
	
		// Start a goroutine connecting to the server asynchronously.
		go ac.resetTransport()
		return nil
	}

通过 go ac.resetTransport() 这一行可以看到 connect 方法新起协程异步去与 server 建立连接。resetTransport 方法中有一行调用了 tryAllAddrs 方法，如下：

		newTr, addr, reconnect, err := ac.tryAllAddrs(addrs, connectDeadline)

我们猜测是在这个方法中去轮询 address 与 每个 address 的 server 建立连接。

	func (ac *addrConn) tryAllAddrs(addrs []resolver.Address, connectDeadline time.Time) (transport.ClientTransport, resolver.Address, *grpcsync.Event, error) {
		for _, addr := range addrs {
			...
	
			newTr, reconnect, err := ac.createTransport(addr, copts, connectDeadline)
			if err == nil {
				return newTr, addr, reconnect, nil
			}
			ac.cc.blockingpicker.updateConnectionError(err)
		}
	
		// Couldn't connect to any address.
		return nil, resolver.Address{}, nil, fmt.Errorf("couldn't connect to any address")
	}

一看这个方法，果然如此，遍历所有地址，然后调用了 createTransport 方法，为每个地址的服务器建立连接，看到这里，我们也明白了 Stream 的实现。传统的 client 实现是对某个 server 地址发起 connect，Stream 的实质无非是对一批 server 的 address 进行轮询并建立 connect


#### 总结
grpc 负载均衡的实现是通过客户端路由的方式，先通过服务发现获取一个 resolver.Address 列表，resolver.Address 中包含了服务器地址和负载均衡服务名字，通过这个名字去初始化响应的 balancer，dns_resolver 中默认是使用的 grpclb 这个负载均衡器，寻址方式是轮询。通过调用 picker 去 生成一个 SubConn，SubConn 中包括服务器的地址列表，采用异步的方式对地址列表进行轮询，然后为每一个服务端地址都进行 connect 。

其实关于 resolver、balancer、picker 的实现还有很多细节部分，这里等待后续研究。
