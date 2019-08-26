
### grpc hello world  client 解析

上一节我们介绍了 grpc 输出 hello world 过程中 server 监听和处理请求的过程。这一节中我们将介绍 client 发出请求的过程。

来先看代码：

	func main() {
		// Set up a connection to the server.
		conn, err := grpc.Dial(address, grpc.WithInsecure())
		if err != nil {
			log.Fatalf("did not connect: %v", err)
		}
		defer conn.Close()
		c := pb.NewGreeterClient(conn)
	
		// Contact the server and print out its response.
		name := defaultName
		if len(os.Args) > 1 {
			name = os.Args[1]
		}
		ctx, cancel := context.WithTimeout(context.Background(), time.Second)
		defer cancel()
		r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
		if err != nil {
			log.Fatalf("could not greet: %v", err)
		}
		log.Printf("Greeting: %s", r.Message)
	}

可以看到 client 的建立也可以大致分为 3 步：

1）创建一个客户端连接 conn

2）通过一个 conn 创建一个客户端

3）发起 rpc 调用

ok，那我们开始 step by step ，具体看看每一步做了啥

#### 1）创建一个客户端连接 conn

	conn, err := grpc.Dial(address, grpc.WithInsecure())

通过 Dial 方法创建 conn，Dial 调用了 DialContext 方法

	func Dial(target string, opts ...DialOption) (*ClientConn, error) {
		return DialContext(context.Background(), target, opts...)
	}

跟进 DialContext，发现 DialContext 这个方法非常长，这里就不贴代码了，具体就是先实例化了一个 ClientConn 的结构体，然后主要为 ClientConn 的 dopts 的各个属性进行初始化赋值。

	cc := &ClientConn{
			target:            target,
			csMgr:             &connectivityStateManager{},
			conns:             make(map[*addrConn]struct{}),
			dopts:             defaultDialOptions(),
			blockingpicker:    newPickerWrapper(),
			czData:            new(channelzData),
			firstResolveEvent: grpcsync.NewEvent(),
	}

我们先来看看 ClientConn 的结构


	type ClientConn struct {
		ctx    context.Context
		cancel context.CancelFunc
	
		target       string
		parsedTarget resolver.Target
		authority    string
		dopts        dialOptions
		csMgr        *connectivityStateManager
	
		balancerBuildOpts balancer.BuildOptions
		blockingpicker    *pickerWrapper
	
		mu              sync.RWMutex
		resolverWrapper *ccResolverWrapper
		sc              *ServiceConfig
		conns           map[*addrConn]struct{}
		// Keepalive parameter can be updated if a GoAway is received.
		mkp             keepalive.ClientParameters
		curBalancerName string
		balancerWrapper *ccBalancerWrapper
		retryThrottler  atomic.Value
	
		firstResolveEvent *grpcsync.Event
	
		channelzID int64 // channelz unique identification number
		czData     *channelzData
	}

dialOptions 其实就是对客户端属性的一些设置，包括压缩解压缩、是否需要认证、超时时间、是否重试等信息。

这里我们来看一下初始化了哪些属性：

####connectivityStateManager

	type connectivityStateManager struct {
		mu         sync.Mutex
		state      connectivity.State
		notifyChan chan struct{}
		channelzID int64
	}

连接的状态管理器，每个连接具有 “IDLE”、“CONNECTING”、“READY”、“TRANSIENT_FAILURE”、“SHUTDOW
N”、“Invalid-State” 这几种状态。


#### pickerWrapper

	type pickerWrapper struct {
		mu         sync.Mutex
		done       bool
		blockingCh chan struct{}
		picker     balancer.Picker
	
		// The latest connection happened.
		connErrMu sync.Mutex
		connErr   error
	}

pickerWrapper 是对 balancer.Picker 的一层封装，balancer.Picker 其实是一个负载均衡器，它里面只有一个 Pick 方法，它返回一个 SubConn 连接。

	type Picker interface {
		Pick(ctx context.Context, opts PickOptions) (conn SubConn, done func(DoneInfo), err error)
	}

什么是 SubConn 呢？看一下这个类的介绍

	// Each sub connection contains a list of addresses. gRPC will
	// try to connect to them (in sequence), and stop trying the
	// remainder once one connection is successful.

这里我们就明白了，在分布式环境下，可能会存在多个 client 和 多个 server，client 发起一个 rpc 调用之前，需要通过 balancer 去找到一个 server 的 address，balancer 的 Picker 类返回一个 SubConn，SubConn 里面包含了多个 server 的 address，假如返回的 SubConn 是 “READY” 状态，grpc 会发送 RPC 请求，否则则会阻塞，等待 UpdateBalancerState 这个方法更新连接的状态并且通过 picker 获取一个新的 SubConn 连接。

#### channelz

channelz 主要用来监测 server 和 channel 的状态，这里的概念和实现比较复杂，暂时不进行深入研究，感兴趣的同学可以参考：https://github.com/grpc/proposal/blob/master/A14-channelz.md 这个 proposal ，初始化的代码如下：

	if channelz.IsOn() {
			if cc.dopts.channelzParentID != 0 {
				cc.channelzID = channelz.RegisterChannel(&channelzChannel{cc}, cc.dopts.channelzParentID, target)
				channelz.AddTraceEvent(cc.channelzID, &channelz.TraceEventDesc{
					Desc:     "Channel Created",
					Severity: channelz.CtINFO,
					Parent: &channelz.TraceEventDesc{
						Desc:     fmt.Sprintf("Nested Channel(id:%d) created", cc.channelzID),
						Severity: channelz.CtINFO,
					},
				})
			} else {
				cc.channelzID = channelz.RegisterChannel(&channelzChannel{cc}, 0, target)
				channelz.AddTraceEvent(cc.channelzID, &channelz.TraceEventDesc{
					Desc:     "Channel Created",
					Severity: channelz.CtINFO,
				})
			}
			cc.csMgr.channelzID = cc.channelzID
		}


#### Authentication

这一段是对认证信息的初始化校验，这里暂不研究，感兴趣的同学可以了解下 ：https://grpc.io/docs/guides/auth/

	if !cc.dopts.insecure {
			if cc.dopts.copts.TransportCredentials == nil && cc.dopts.copts.CredsBundle == nil {
				return nil, errNoTransportSecurity
			}
			if cc.dopts.copts.TransportCredentials != nil && cc.dopts.copts.CredsBundle != nil {
				return nil, errTransportCredsAndBundle
			}
		} else {
			if cc.dopts.copts.TransportCredentials != nil || cc.dopts.copts.CredsBundle != nil {
				return nil, errCredentialsConflict
			}
			for _, cd := range cc.dopts.copts.PerRPCCredentials {
				if cd.RequireTransportSecurity() {
					return nil, errTransportCredentialsMissing
				}
			}
		}
	
#### Dialer

Dialer 是发起 rpc 请求的调用器，dialer 中实现了对 rpc 请求调用的具体细节，所以可以算是我们的重点研究对象之一。dialer 中包括了连接建立、地址解析、服务发现、长连接等等具体策略的实现。

	if cc.dopts.copts.Dialer == nil {
			cc.dopts.copts.Dialer = newProxyDialer(
				func(ctx context.Context, addr string) (net.Conn, error) {
					network, addr := parseDialTarget(addr)
					return (&net.Dialer{}).DialContext(ctx, network, addr)
				},
			)
		}

这一段方法只是简单进行了地址的规则解析，我们具体看 DialContext 方法，其中有一行：

	addrs, err := d.resolver().resolveAddrList(resolveCtx, "dial", network, address, d.LocalAddr)

可以看到通过 dialer 的 resolver 来进行服务发现，这里以后我们再单独详细讲解。

这里值得一提的是，通过 dialContext 可以看出，这里的 dial 有两种请求方式，一种是 dialParallel , 另一种是 dialSerial。dialParallel 发出两个完全相同的请求，采用第一个返回的结果，抛弃掉第二个请求。dialSerial 则是发出一串（多个）请求。然后采取第一个返回的请求结果（ 成功或者失败）。

#### scChan

scChan 是 dialOptions 中的一个属性，定义如下：

	scChan  <-chan ServiceConfig

可以看到其实他是一个 ServiceConfig类型的一个 channel，那么 ServiceConfig 是什么呢？源码中对这个类的介绍如下：

	// ServiceConfig is provided by the service provider and contains parameters for how clients that connect to the service should behave.

通过介绍得知 ServiceConfig 是服务提供方约定的一些参数。这里说明 client 提供给 server 一个可以通过 channel 来修改这些参数的入口。这里到时候我们介绍服务发现时可以细讲，我们在这里只需要知道 client 的某些属性是可以被 server 修改的就行了

	if cc.dopts.scChan != nil {
			// Try to get an initial service config.
			select {
			case sc, ok := <-cc.dopts.scChan:
				if ok {
					cc.sc = &sc
					scSet = true
				}
			default:
			}
		}

#### 2）通过一个 conn 创建一个客户端

通过一个 conn 创建客户端的代码如下：

		c := pb.NewGreeterClient(conn)

这一步非常简单，其实是 pb 文件中生成的代码，就是创建一个 greeterClient 的客户端。

		type greeterClient struct {
			cc *grpc.ClientConn
		}
	
		func NewGreeterClient(cc *grpc.ClientConn) GreeterClient {
			return &greeterClient{cc}
		}

#### 3）发起 rpc 调用

前面在创建 Dialer 的时候，我们已经将请求的 target 解析成了 address。我们猜这一步应该是向指定 address 发起 rpc 请求了。来具体看看

	func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
		out := new(HelloReply)
		err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)
		if err != nil {
			return nil, err
		}
		return out, nil
	}

SayHello 方法是通过调用 Invoke 的方法去发起 rpc 调用， Invoke 方法如下：

	func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
		// allow interceptor to see all applicable call options, which means those
		// configured as defaults from dial option as well as per-call options
		opts = combine(cc.dopts.callOptions, opts)
	
		if cc.dopts.unaryInt != nil {
			return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
		}
		return invoke(ctx, method, args, reply, cc, opts...)
	}

	func invoke(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error {
		cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
		if err != nil {
			return err
		}
		if err := cs.SendMsg(req); err != nil {
			return err
		}
		return cs.RecvMsg(reply)
	}

Invoke 方法调用了 invoke， 在 invoke 这个方法里面，果然不出所料，我们看到了 sendMsg 和 recvMsg 接口，这两个接口在 clientStream 中被实现了。

#### SendMsg

我们先来看看 clientStream 中定义的 sendMsg，关键代码如下：

	func (cs *clientStream) SendMsg(m interface{}) (err error) {
		
		...
	
		// load hdr, payload, data
		hdr, payload, data, err := prepareMsg(m, cs.codec, cs.cp, cs.comp)
		if err != nil {
			return err
		}
	
		...
	
		op := func(a *csAttempt) error {
			err := a.sendMsg(m, hdr, payload, data)
			// nil out the message and uncomp when replaying; they are only needed for
			// stats which is disabled for subsequent attempts.
			m, data = nil, nil
			return err
		}
		
	}

先准备数据，然后再调用 csAttempt 这个结构体中的 sendMsg 方法，

	func (a *csAttempt) sendMsg(m interface{}, hdr, payld, data []byte) error {
		cs := a.cs
		if a.trInfo != nil {
			a.mu.Lock()
			if a.trInfo.tr != nil {
				a.trInfo.tr.LazyLog(&payload{sent: true, msg: m}, true)
			}
			a.mu.Unlock()
		}
		if err := a.t.Write(a.s, hdr, payld, &transport.Options{Last: !cs.desc.ClientStreams}); err != nil {
			if !cs.desc.ClientStreams {
				// For non-client-streaming RPCs, we return nil instead of EOF on error
				// because the generated code requires it.  finish is not called; RecvMsg()
				// will call it with the stream's status independently.
				return nil
			}
			return io.EOF
		}
		if a.statsHandler != nil {
			a.statsHandler.HandleRPC(cs.ctx, outPayload(true, m, data, payld, time.Now()))
		}
		if channelz.IsOn() {
			a.t.IncrMsgSent()
		}
		return nil
	}

最终是通过 a.t.Write 发出的数据写操作，a.t 是一个 ClientTransport 类型，所以最终是通过 ClientTransport 这个结构体的 Write 方法发送数据

#### RecvMsg

发送数据是通过 ClientTransport 的 Write 方法，我们猜测接收数据肯定是某个结构体的 Read 方法。这里我们来详细看一下

	func (a *csAttempt) recvMsg(m interface{}, payInfo *payloadInfo) (err error) {
		
		...
		
		err = recv(a.p, cs.codec, a.s, a.dc, m, *cs.callInfo.maxReceiveMessageSize, payInfo, a.decomp)
		
		...
		
		if a.statsHandler != nil {
			a.statsHandler.HandleRPC(cs.ctx, &stats.InPayload{
				Client:   true,
				RecvTime: time.Now(),
				Payload:  m,
				// TODO truncate large payload.
				Data:       payInfo.uncompressedBytes,
				WireLength: payInfo.wireLength,
				Length:     len(payInfo.uncompressedBytes),
			})
		}
		
		...
	}

可以看到调用了 recv 方法：

	func recv(p *parser, c baseCodec, s *transport.Stream, dc Decompressor, m interface{}, maxReceiveMessageSize int, payInfo *payloadInfo, compressor encoding.Compressor) error {
		d, err := recvAndDecompress(p, s, dc, maxReceiveMessageSize, payInfo, compressor)
		...
	}

再看 recvAndDecompress 方法，调用了 recvMsg

	func recvAndDecompress(p *parser, s *transport.Stream, dc Decompressor, maxReceiveMessageSize int, payInfo *payloadInfo, compressor encoding.Compressor) ([]byte, error) {
		pf, d, err := p.recvMsg(maxReceiveMessageSize)
		...
	}

	func (p *parser) recvMsg(maxReceiveMessageSize int) (pf payloadFormat, msg []byte, err error) {
		if _, err := p.r.Read(p.header[:]); err != nil {
			return 0, nil, err
		}
	
		...
	}

这里比较清楚了，最终还是调用了 p.r.Read 方法，p.r 是一个 io.Reader 类型。果然万变不离其中，最终都是要落到 IO 上。

到这里，整个 client 结构已经基本解析清楚了，but wait，总感觉哪里不太对，接收数据是调用 io.Reader ，按道理发送数据应该也是调用 io.Writer 才对。可是追溯到 ClientTransport 这里，发现它是一个 interface ，并没有实现 Write 方法，所以，Write 也是一个接口，这里是不是可以继续追溯呢？

		Write(s *Stream, hdr []byte, data []byte, opts *Options) error
		
		
返回去从头看，我们找到了 transport 的来源，在 Serve() 方法 的 handleRawConn 方法中，newHttp2Transport，创建了一个 Http2Transport ，然后通过 serveStreams 方法将这个 Http2Transport 层层透传下去。


	// Finish handshaking (HTTP2)
	st := s.newHTTP2Transport(conn, authInfo)
	if st == nil {
		return
	}

	rawConn.SetDeadline(time.Time{})
	if !s.addConn(st) {
		return
	}
	go func() {
		s.serveStreams(st)
		s.removeConn(st)
	}()


继续看一下 http2Client 的 Write 方法，如下：


	func (t *http2Server) Write(s *Stream, hdr []byte, data []byte, opts *Options) error {
		...

		hdr = append(hdr, data[:emptyLen]...)
		data = data[emptyLen:]
		df := &dataFrame{
			streamID:    s.id,
			h:           hdr,
			d:           data,
			onEachWrite: t.setResetPingStrikes,
		}
		if err := s.wq.get(int32(len(hdr) + len(data))); err != nil {
			select {
			case <-t.ctx.Done():
				return ErrConnClosing
			default:
			}
			return ContextErr(s.ctx.Err())
		}
		return t.controlBuf.put(df)
	}
	

可以看到，最终是把 data 放到了一个 controlBuf 的结构体里面


	// controlBuf delivers all the control related tasks (e.g., window
	// updates, reset streams, and various settings) to the controller.
	controlBuf *controlBuffer

controlBuf 是 http2 客户端发送数据的实现，这里留待后续研究

