> 本文原发表于：https://diu.life/lessons/grpc-read/grpc-auth-oauth2/
> 最新版本请访问原文链接

## grpc 认证鉴权 —— oauth2

前面我们说了 tls 认证，tls 保证了 client 和 server 通信的安全性，但是无法做到接口级别的权限控制。例如有 A、B、C、D 四个系统，存在下面两个场景：
1、我们希望 A 可以访问 B、C 系统，但是不能访问 D 系统
2、B 系统提供了 b1、b2、b3 三个接口，我们希望 A 系统可以访问 b1、b2 接口，但是不能访问 b3 接口。
此时 tls 认证肯定是无法实现上面两个诉求的，对于这两个场景，grpc 提供了 oauth2 的认证方式。对 oauth2 不了解的同学可以参考 http://www.ruanyifeng.com/blog/2019/04/oauth_design.html

### oauth2 认证鉴权实现
grpc 官方提供了对 oauth2 认证鉴权的实现 demo，放在 examples 目录的 features 目录的 authentication 目录下，我们来看一下源码实现

#### server
server 端源码实现如下：

	func main() {
		flag.Parse()
		fmt.Printf("server starting on port %d...\n", *port)
	
		cert, err := tls.LoadX509KeyPair(testdata.Path("server1.pem"), testdata.Path("server1.key"))
		if err != nil {
			log.Fatalf("failed to load key pair: %s", err)
		}
		opts := []grpc.ServerOption{
			// The following grpc.ServerOption adds an interceptor for all unary
			// RPCs. To configure an interceptor for streaming RPCs, see:
			// https://godoc.org/google.golang.org/grpc#StreamInterceptor
			grpc.UnaryInterceptor(ensureValidToken),
			// Enable TLS for all incoming connections.
			grpc.Creds(credentials.NewServerTLSFromCert(&cert)),
		}
		s := grpc.NewServer(opts...)
		ecpb.RegisterEchoServer(s, &ecServer{})
		lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
		if err != nil {
			log.Fatalf("failed to listen: %v", err)
		}
		if err := s.Serve(lis); err != nil {
			log.Fatalf("failed to serve: %v", err)
		}
	}

server 端先调用了 tls 包下的 LoadX509KeyPair，通过 server 的公钥和私钥生成了一个 Certificate 结构体来保存证书信息。然后注册了一个校验 token 的方法到拦截器中，并将证书信息设置到 serverOption 中，构造 server 的时候层层透传进去，最终会被设置到 Server 里面 ServerOptions 结构中的 credentials.TransportCredentials 和 UnaryServerInterceptor 中。

我们来看看这两个结构什么时候会被调用，先梳理调用链路，在 s.Serve ——> s.handleRawConn ——> s.serveStreams ——> s.handleStream ——> s.processUnaryRPC 方法中有一行

	reply, appErr := md.Handler(srv.server, ctx, df, s.opts.unaryInt)

可以看到调用了 md.Handler 方法，将 s.opts.unaryInt 这个结构传入了进去。s.opts.unaryInt 就是我们之前注册的 UnaryServerInterceptor 拦截器。md 是一个 MethodDesc 这个结构，包括了 MethodName 和 Handler

	type MethodDesc struct {
		MethodName string
		Handler    methodHandler
	}

这里会取出我们之前注册进去的结构，还记得我们介绍 helloworld 时 RegisterService 吗？至于如何取出 MethodName，源码中的设计非常复杂，经过了层层包装，这里不是本节重点就不赘述了。

	func RegisterGreeterServer(s *grpc.Server, srv GreeterServer) {
		s.RegisterService(&_Greeter_serviceDesc, srv)
	}
	
	var _Greeter_serviceDesc = grpc.ServiceDesc{
		ServiceName: "helloworld.Greeter",
		HandlerType: (*GreeterServer)(nil),
		Methods: []grpc.MethodDesc{
			{
				MethodName: "SayHello",
				Handler:    _Greeter_SayHello_Handler,
			},
		},
		Streams:  []grpc.StreamDesc{},
		Metadata: "helloworld.proto",
	}

我们看到 md.Handler 其实是 _Greeter_SayHello_Handler 这个结构，它也是在 pb 文件中生成的。

	func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
		in := new(HelloRequest)
		if err := dec(in); err != nil {
			return nil, err
		}
		if interceptor == nil {
			return srv.(GreeterServer).SayHello(ctx, in)
		}
		info := &grpc.UnaryServerInfo{
			Server:     srv,
			FullMethod: "/helloworld.Greeter/SayHello",
		}
		handler := func(ctx context.Context, req interface{}) (interface{}, error) {
			return srv.(GreeterServer).SayHello(ctx, req.(*HelloRequest))
		}
		return interceptor(ctx, in, info, handler)
	}

这里调用了我们传入的 interceptor 方法。回到我们的调用：

	reply, appErr := md.Handler(srv.server, ctx, df, s.opts.unaryInt)

所以其实是调用了 s.opts.unaryInt 这个拦截器。这个拦截器是我们之前在 创建 server 的时候赋值的。

		opts := []grpc.ServerOption{
			// The following grpc.ServerOption adds an interceptor for all unary
			// RPCs. To configure an interceptor for streaming RPCs, see:
			// https://godoc.org/google.golang.org/grpc#StreamInterceptor
			grpc.UnaryInterceptor(ensureValidToken),
			// Enable TLS for all incoming connections.
			grpc.Creds(credentials.NewServerTLSFromCert(&cert)),
		}
		s := grpc.NewServer(opts...)

看 grpc.UnaryInterceptor 这个方法，其实是将 ensureValidToken 这个函数赋值给了 s.opts.unaryInt

		func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {
			return newFuncServerOption(func(o *serverOptions) {
				if o.unaryInt != nil {
					panic("The unary server interceptor was already set and may not be reset.")
				}
				o.unaryInt = i
			})
		}

所以之前我们执行的这一行

		return interceptor(ctx, in, info, handler)

其实是执行了 ensureValidToken 这个函数，这个函数就是我们在 server 端定义的 token 校验的函数。先取出我们传入的 metadata 数据，然后校验 token

		func ensureValidToken(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
			md, ok := metadata.FromIncomingContext(ctx)
			if !ok {
				return nil, errMissingMetadata
			}
			// The keys within metadata.MD are normalized to lowercase.
			// See: https://godoc.org/google.golang.org/grpc/metadata#New
			if !valid(md["authorization"]) {
				return nil, errInvalidToken
			}
			// Continue execution of handler after ensuring a valid token.
			return handler(ctx, req)
	}

校验完 token 后，最终执行了 handler(ctx, req) 

		handler := func(ctx context.Context, req interface{}) (interface{}, error) {
			return srv.(GreeterServer).SayHello(ctx, req.(*HelloRequest))
		}
		return interceptor(ctx, in, info, handler)

可以看到最终其实执行了 GreeterServer 的 SayHello 这个函数，也就是我们在 main 函数中定义的，这个函数就是我们在 server 端定义的提供 SayHello 给客户端回消息的函数。

	// SayHello implements helloworld.GreeterServer
	func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
		log.Printf("Received: %v", in.Name)
		return &pb.HelloReply{Message: "Hello " + in.Name}, nil
	}

这里还可以额外说一下，md.Handler 执行完之后，其实 reply 就是 SayHello 的回包。

		reply, appErr := md.Handler(srv.server, ctx, df, s.opts.unaryInt)

获取到回包之后 server 执行了 sendResponse 方法，将回包发送给 client，这个方法我们之前已经剖析过了，最终会调用 http2Server 的 Write 方法。

	if err := s.sendResponse(t, stream, reply, cp, opts, comp); err != nil {

看到这里，server 端对 token 的校验在哪里执行的我们已经清楚了。假如还没有被绕晕，那么恭喜你！可以继续完成 client 的挑战了。

#### client
在 client 中，先看 main 函数
	
	// Set up the credentials for the connection.
		perRPC := oauth.NewOauthAccess(fetchToken())
		creds, err := credentials.NewClientTLSFromFile(testdata.Path("ca.pem"), "x.test.youtube.com")
		if err != nil {
			log.Fatalf("failed to load credentials: %v", err)
		}
		opts := []grpc.DialOption{
			// In addition to the following grpc.DialOption, callers may also use
			// the grpc.CallOption grpc.PerRPCCredentials with the RPC invocation
			// itself.
			// See: https://godoc.org/google.golang.org/grpc#PerRPCCredentials
			grpc.WithPerRPCCredentials(perRPC),
			// oauth.NewOauthAccess requires the configuration of transport
			// credentials.
			grpc.WithTransportCredentials(creds),
		}
	
		conn, err := grpc.Dial(*addr, opts...)

可以看到 client 首先通过 NewOauthAccess 方法生成了包含 token 信息的 PerRPCCredentials 结构

	func NewOauthAccess(token *oauth2.Token) credentials.PerRPCCredentials {
		return oauthAccess{token: *token}
	}

然后再将 PerRPCCredentials 通过  grpc.WithPerRPCCredentials(perRPC) 添加到了到了 client 的 DialOptions 中的 transport.ConnectOptions 结构中的 [] credentials.PerRPCCredentials 结构中。

那么这个结构什么时候被使用呢，我们来看看。先梳理下调用链 ，在 client 调用的 Invoke ——> invoke ——> newClientStream ——> cs.newAttemptLocked ——> cs.cc.getTransport ——> pick ——> acw.getAddrConn().getReadyTransport() ——>  ac.connect() ——> ac.resetTransport() ——> ac.tryAllAddrs ——> ac.createTransport ——> transport.NewClientTransport ——> newHTTP2Client  这个方法里面，有这么一段代码，先取出 []credentials.PerRPCCredentials 中的所有 PerRPCCredentials 添加到 perRPCCreds 中。

		transportCreds := opts.TransportCredentials
		perRPCCreds := opts.PerRPCCredentials
	
		if b := opts.CredsBundle; b != nil {
			if t := b.TransportCredentials(); t != nil {
				transportCreds = t
			}
			if t := b.PerRPCCredentials(); t != nil {
				perRPCCreds = append(perRPCCreds, t)
			}
		}

然后再将 perRPCCreds 赋值给 http2Client 的 perRPCCreds 属性

	t := &http2Client{

		...

		perRPCCreds:           perRPCCreds,
		
		...
	}

那么 perRPCCreds 属性什么时候被用呢？来继续跟踪，newClientStream  方法中有一段代码

		op := func(a *csAttempt) error { return a.newStream() }
		if err := cs.withRetry(op, func() { cs.bufferForRetryLocked(0, op) }); err != nil {
			cs.finish(err)
			return nil, err
		}

这里调用了 csAttempt 的 newStream ——> a.t.NewStream (http2Client 的 NewStream) ——> createHeaderFields ——> getTrAuthData 方法

	func (t *http2Client) getTrAuthData(ctx context.Context, audience string) (map[string]string, error) {
		if len(t.perRPCCreds) == 0 {
			return nil, nil
		}
		authData := map[string]string{}
		for _, c := range t.perRPCCreds {
			data, err := c.GetRequestMetadata(ctx, audience)
			if err != nil {
				if _, ok := status.FromError(err); ok {
					return nil, err
				}
	
				return nil, status.Errorf(codes.Unauthenticated, "transport: %v", err)
			}
			for k, v := range data {
				// Capital header names are illegal in HTTP/2.
				k = strings.ToLower(k)
				authData[k] = v
			}
		}
		return authData, nil
	}
	
这个方法，通过调用 GetRequestMetadata 取出 token 信息，这里会调用 oauth 的 GetRequestMetadata 方法 ，按照指定格式拼装成一个  map[string]string{} 的形式
	
	func (s *serviceAccount) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
		s.mu.Lock()
		defer s.mu.Unlock()
		if !s.t.Valid() {
			var err error
			s.t, err = s.config.TokenSource(ctx).Token()
			if err != nil {
				return nil, err
			}
		}
		return map[string]string{
			"authorization": s.t.Type() + " " + s.t.AccessToken,
		}, nil
	}

然后将以 map[string]string{} 的形式组装成一个 string map 返回，如下：

	   for k, v := range authData {
			headerFields = append(headerFields, hpack.HeaderField{Name: k, Value: encodeMetadataHeader(k, v)})
		}

返回的 map 会被遍历每个 key，并设置到 headerFields 中，以 http 头部的形式发送出去。数据最终会被 metadata.FromIncomingContext(ctx) 获取到，然后被取出 map 数据。


至此，client 和 server 的数据流转过程被打通
