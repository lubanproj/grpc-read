
### grpc hello world server 解析

我们介绍 grpc quick start 时，通过快速启动一个 grpc server 端和 client 端，然后以 rpc 调用的方式输出一个 hello world。那么输出 hello world 需要经过哪些方法的处理呢？.......这个我也不知道，所以我们先去瞅瞅源码，探究一下 hello world 的背后是连接是如何建立的，然后一起来解读这个问题哈哈。

这节内容我们先来研究一下 server 端连接建立过程。

先放上 server 端的 main 函数。

	func main() {
		lis, err := net.Listen("tcp", port)
		if err != nil {
			log.Fatalf("failed to listen: %v", err)
		}
		s := grpc.NewServer()
		pb.RegisterGreeterServer(s, &server{})
		if err := s.Serve(lis); err != nil {
			log.Fatalf("failed to serve: %v", err)
		}
	}

我们发现其实 server 端连接的建立主要包括三步：

（1）创建 server

（2）server 的注册

（3）调用 Serve 监听端口并处理请求

ok，弄清楚主流程之后下面我们进入每个步骤里面去看一下代码实现。

#### 1、创建  server

server 的创建比较简单，其实就下面一个方法：

	func NewServer(opt ...ServerOption) *Server {
		opts := defaultServerOptions
		for _, o := range opt {
			o.apply(&opts)
		}
		s := &Server{
			lis:    make(map[net.Listener]bool),
			opts:   opts,
			conns:  make(map[transport.ServerTransport]bool),
			m:      make(map[string]*service),
			quit:   grpcsync.NewEvent(),
			done:   grpcsync.NewEvent(),
			czData: new(channelzData),
		}
		s.cv = sync.NewCond(&s.mu)
		if EnableTracing {
			_, file, line, _ := runtime.Caller(1)
			s.events = trace.NewEventLog("grpc.Server", fmt.Sprintf("%s:%d", file, line))
		}
	
		if channelz.IsOn() {
			s.channelzID = channelz.RegisterServer(&channelzServer{s}, "")
		}
		return s
	}
这个方法的核心无非是创建了一个 server 结构体，然后为结构体的属性赋值。我们顺便来瞅瞅 server 的结构：

	// Server is a gRPC server to serve RPC requests.
	type Server struct {
		// serverOptions 就是描述协议的各种参数选项，包括发送和接收的消息大小、buffer大小等等各种，跟 http Headers 类似，我们这里就暂时先不管
		opts serverOptions
	
		// 一个互斥锁
		mu     sync.Mutex // guards following
		// listener map
		lis    map[net.Listener]bool
		// connections map
		conns  map[transport.ServerTransport]bool
		// server 是否在处理请求的一个状态位
		serve  bool
		drain  bool
		cv     *sync.Cond          // signaled when connections close for GracefulStop
		// service map
		m      map[string]*service // service name -> service info
		events trace.EventLog
	
		quit               *grpcsync.Event
		done               *grpcsync.Event
		channelzRemoveOnce sync.Once
		serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop
	
		channelzID int64 // channelz unique identification number
		czData     *channelzData
	}

虽然 server 结构体里面各种乱起八糟的字段，但是我们可以先不管哈哈哈，比较重要的无非就是三个 map 表分别用来存放多个 listener 、connection 和 service。其他字段都是为了实现协议描述或者并发控制的功能。我们重点关注下 

	m      map[string]*service 

这个结构，service 中主要包含了 MethodDesc 和 StreamDesc 这两个 map

	type service struct {
		server interface{} // the server for service methods
		md     map[string]*MethodDesc
		sd     map[string]*StreamDesc
		mdata  interface{}
	}

![enter image description here](https://images.gitbook.cn/2c36f9d0-b69a-11e9-8dd9-33673ef07123)


####2、server 注册

server 的注册调用了 RegisterGreeterServer 方法，这个方法是 pb.go 文件里面的，如下：

	func RegisterGreeterServer(s *grpc.Server, srv GreeterServer) {
		s.RegisterService(&_Greeter_serviceDesc, srv)
	}

这个方法调用了 server 的 RegisterService 方法，然后传入了一个  ServiceDesc 的数据结构，如下 ：

	var _Greeter_serviceDesc = grpc.ServiceDesc{
		ServiceName: "helloworld.Greeter",
		HandlerType: (*GreeterServer)(nil),
		Methods: []grpc.MethodDesc{
			{
				MethodName: "SayHello",
				Handler:    _Greeter_SayHello_Handler,
			},
			{
				MethodName: "SayHelloAgain",
				Handler:    _Greeter_SayHelloAgain_Handler,
			},
		},
		Streams:  []grpc.StreamDesc{},
		Metadata: "helloworld.proto",
	}

我们来看看 RegisterService 这个方法，可以看到主要是调用了 register 方法，register 方法则按照方法名为 key，将方法注入到 server 的 service map 中。看到这里我们其实可以预测一下，server 不同 rpc 请求的处理，也是根据 service 中不同的 serviceName 去 service map 中取出不同的 handler 进行处理

	func (s *Server) RegisterService(sd *ServiceDesc, ss interface{}) {
		ht := reflect.TypeOf(sd.HandlerType).Elem()
		st := reflect.TypeOf(ss)
		if !st.Implements(ht) {
			grpclog.Fatalf("grpc: Server.RegisterService found the handler of type %v that does not satisfy %v", st, ht)
		}
		s.register(sd, ss)
	}
	
	func (s *Server) register(sd *ServiceDesc, ss interface{}) {
		s.mu.Lock()
		defer s.mu.Unlock()
		s.printf("RegisterService(%q)", sd.ServiceName)
		if s.serve {
			grpclog.Fatalf("grpc: Server.RegisterService after Server.Serve for %q", sd.ServiceName)
		}
		if _, ok := s.m[sd.ServiceName]; ok {
			grpclog.Fatalf("grpc: Server.RegisterService found duplicate service registration for %q", sd.ServiceName)
		}
		srv := &service{
			server: ss,
			md:     make(map[string]*MethodDesc),
			sd:     make(map[string]*StreamDesc),
			mdata:  sd.Metadata,
		}
		for i := range sd.Methods {
			d := &sd.Methods[i]
			srv.md[d.MethodName] = d
		}
		for i := range sd.Streams {
			d := &sd.Streams[i]
			srv.sd[d.StreamName] = d
		}
		s.m[sd.ServiceName] = srv
	}

####3、Serve 过程

回想所有 C/S 模式下，client 和 server 的通信基本是类似的。大致过程无非是 server 通过死循环的方式在某一个端口实现监听，然后 client 对这个端口发起连接请求，握手成功后建立连接，然后 server 处理 client 发送过来的请求数据，根据请求类型和请求参数，调用不同的 handler 进行处理，回写响应数据。

所以，对 server 端来说，主要是了解其如何实现监听，如何为请求分配不同的 handler 和 回写响应数据。

上面我们得知 server 调用了 Serve 方法来进行处理，所以立马就想跟进去看看。
跳过前面一堆条件检查和控制代码，直接锁定了一个 for 循环，如下：（中间的一些代码已省略）


	for {
			rawConn, err := lis.Accept()
			
			......
	
			s.serveWG.Add(1)
			go func() {
				s.handleRawConn(rawConn)
				s.serveWG.Done()
			}()
		}

ok，我们已经看到了监听过程，server 的监听果然是通过一个死循环 调用了 lis.Accept() 进行端口监听。

继续往下看，我们发现新起协程调用了 handleRawConn 这个方法，为了节约篇幅，我们直接看重点代码，如下：

	func (s *Server) handleRawConn(rawConn net.Conn) {
		
		...
	
		conn, authInfo, err := s.useTransportAuthenticator(rawConn)
		
		...
	
		// Finish handshaking (HTTP2)
		st := s.newHTTP2Transport(conn, authInfo)
		if st == nil {
			return
		}
		...
		
		go func() {
			s.serveStreams(st)
			s.removeConn(st)
		}()
	}

可以看到 handleRawConn 里面实现了 http 的 handshake，还记得之前我们说过，grpc 是基于 http2 实现的吗？这里是不是实锤了....... 发现又通过一个新的协程调用了 serveStreams 这个方法，这个方法干了啥呢？

	func (s *Server) serveStreams(st transport.ServerTransport) {
		defer st.Close()
		var wg sync.WaitGroup
		st.HandleStreams(func(stream *transport.Stream) {
			wg.Add(1)
			go func() {
				defer wg.Done()
				s.handleStream(st, stream, s.traceInfo(st, stream))
			}()
		}, func(ctx context.Context, method string) context.Context {
			if !EnableTracing {
				return ctx
			}
			tr := trace.New("grpc.Recv."+methodFamily(method), method)
			return trace.NewContext(ctx, tr)
		})
		wg.Wait()
	}

其实它主要调用了 handleStream ，继续跟进 handleStream 方法，我们发现了重要线索，如下（省略了部分无关代码）

	func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
		sm := stream.Method()
		
		...
	
		service := sm[:pos]
		method := sm[pos+1:]
	
		srv, knownService := s.m[service]
		if knownService {
			if md, ok := srv.md[method]; ok {
				s.processUnaryRPC(t, stream, srv, md, trInfo)
				return
			}
			if sd, ok := srv.sd[method]; ok {
				s.processStreamingRPC(t, stream, srv, sd, trInfo)
				return
			}
		}
		
		...
	}
	
	
重要线索就是这一行
		
		srv, knownService := s.m[service]

还记得我们之前的预测吗？根据 serviceName 去 server 中的 service map，也就是 m 这个字段，里面去取出 handler 进行处理。我们 hello world 这个 demo 的请求不涉及到 stream ，所以直接取出 handler ，然后传给 processUnaryRPC 这个方法进行处理。


	if md, ok := srv.md[method]; ok {
		s.processUnaryRPC(t, stream, srv, md, trInfo)
		return
	}

再来看看 processUnaryRpc 这个方法

	func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, md *MethodDesc, trInfo *traceInfo) (err error) {
			
		...
	
		sh := s.opts.statsHandler
		if sh != nil {
			beginTime := time.Now()
			begin := &stats.Begin{
				BeginTime: beginTime,
			}
			sh.HandleRPC(stream.Context(), begin)
			defer func() {
				end := &stats.End{
					BeginTime: beginTime,
					EndTime:   time.Now(),
				}
				if err != nil && err != io.EOF {
					end.Error = toRPCErr(err)
				}
				sh.HandleRPC(stream.Context(), end)
			}()
		}
		
		...
	
		if err := s.sendResponse(t, stream, reply, cp, opts, comp); err != nil {
			if err == io.EOF {
				// The entire stream is done (for unary RPC only).
				return err
			}
			if s, ok := status.FromError(err); ok {
				if e := t.WriteStatus(stream, s); e != nil {
					grpclog.Warningf("grpc: Server.processUnaryRPC failed to write status: %v", e)
				}
			} else {
				switch st := err.(type) {
				case transport.ConnectionError:
					// Nothing to do here.
				default:
					panic(fmt.Sprintf("grpc: Unexpected error (%T) from sendResponse: %v", st, st))
				}
			}
			if binlog != nil {
				h, _ := stream.Header()
				binlog.Log(&binarylog.ServerHeader{
					Header: h,
				})
				binlog.Log(&binarylog.ServerTrailer{
					Trailer: stream.Trailer(),
					Err:     appErr,
				})
			}
			return err
		}
		
		...
	}

我们终于看到了 handler 对 rpc 的处理：

		sh := s.opts.statsHandler
		sh.HandleRPC(stream.Context(), begin)
		sh.HandleRPC(stream.Context(), end)

同时也看到了 response 的回写

	s.sendResponse(t, stream, reply, cp, opts, comp)

至此，server 端我们的目标实现，追踪到了整个请求和监听、handler 处理 和 response 回写的过程。

		


