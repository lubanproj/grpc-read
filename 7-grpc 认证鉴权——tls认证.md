> 本文原发表于：https://diu.life/lessons/grpc-read/grpc-auth-tls/
> 最新版本请访问原文链接

## grpc 认证鉴权
在了解 grpc 认证鉴权之前，我们有必要先梳理一下认证鉴权方面的知识。

### 1、单体模式下的认证鉴权

在单体模式下，整个应用是一个进程，应用一般只需要一个统一的安全认证模块来实现用户认证鉴权。例如用户登陆时，安全模块验证用户名和密码的合法性。假如合法，为用户生成一个唯一的 Session。将 SessionId 返回给客户端，客户端一般将 SessionId 以 Cookie 的形式记录下来，并在后续请求中传递 Cookie 给服务端来验证身份。为了避免 Session Id被第三者截取和盗用，客户端和应用之前应使用 TLS 加密通信，session 也会设置有过期时间。
	
客户端访问服务端时，服务端一般会用一个拦截器拦截请求，取出 session id，假如 id 合法，则可判断客户端登陆。然后查询用户的权限表，判断用户是否具有执行某次操作的权限。

### 2、微服务模式下的认证鉴权
在微服务模式下，一个整体的应用可能被拆分为多个微服务，之前只有一个服务端，现在会存在多个服务端。对于客户端的单个请求，为保证安全，需要跟每个微服务都要重复上面的过程。这种模式每个微服务都要去实现相同的校验逻辑，肯定是非常冗余的。

#### 用户身份认证
为了避免每个服务端都进行重复认证，采用一个服务进行统一认证。所以考虑一个单点登录的方案，用户只需要登录一次，就可以访问所有微服务。一般在 api 的 gateway 层提供对外服务的入口，所以可以在 api gateway 层提供统一的用户认证。

#### 用户状态保持
由于 http 是一个无状态的协议，前面说到了单体模式下通过 cookie 保存用户状态， cookie 一般存储于浏览器中，用来保存用户的信息。但是 cookie 是有状态的。客户端和服务端在一次会话期间都需要维护 cookie 或者 sessionId，在微服务环境下，我们期望服务的认证是无状态的。所以我们一般采用 token 认证的方式，而非 cookie。

token 由服务端用自己的密钥加密生成，在客户端登录或者完成信息校验时返回给客户端，客户端认证成功后每次向服务端发送请求带上 token，服务端根据密钥进行解密，从而校验 token 的合法，假如合法则认证通过。token 这种方式的校验不需要服务端保存会话状态。方便服务扩展

### 3、grpc 认证鉴权
grpc-go 官方对于认证鉴权的介绍如下：https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-auth-support.md

通过官方介绍可知， grpc-go 认证鉴权是通过 tls + oauth2 实现的。这里不对 tls 和 oauth2 进行详细介绍，假如有不清楚的可以参考阮一峰老师的教程，介绍得比较清楚

tls ：http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html
oauth2 ：http://www.ruanyifeng.com/blog/2019/04/oauth_design.html

下面我们就来具体看看 grpc-go 是如何实现认证鉴权的

grpc-go 官方 doc 说了这里关于 auth 的部分有 demo 放在 examples 目录下的 features 目录下。但是 demo 没有包括证书生成的步骤，这里我们自建一个 demo，从生成证书开始一步步进行 grpc 的认证讲解。

我们先创建一个文件夹 helloauth，然后把之前examples 目录下 helloworld demo 中的 client 和 server 的 go 文件全部 copy 过来，先执行 go mod init helloauth 来生成 go.mod 文件。由于 google.golang.org 被墙，所以执行 go mod edit -replace=google.golang.org/grpc=github.com/grpc/grpc-go@latest， 接着
注意把 替换成 pb "google.golang.org/grpc/examples/helloworld/helloworld"  替换成 pb "helloauth/helloworld" 来引用我们新生成的 pb 文件

#### 生成证书
生成私钥

	openssl ecparam -genkey -name secp384r1 -out server.key

使用私钥生成证书

	openssl req -new -x509 -sha256 -key server.key -out server.pem -days 3650

填写信息（注意 Common Name 要填写服务名）

	Country Name (2 letter code) []:
	State or Province Name (full name) []:
	Locality Name (eg, city) []:
	Organization Name (eg, company) []:
	Organizational Unit Name (eg, section) []:
	Common Name (eg, fully qualified host name) []:helloauth
	Email Address []:

生成完毕后，将证书文件放到 keys 目录下，整个项目目录结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190824164530540.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpdWJyb3RoZXI=,size_16,color_FFFFFF,t_70)
#### 使用证书进行 TLS 通信认证
我们之前的 helloworld demo 中，client 在创建 DialContext 指定非安全模式通信，如下：

		conn, err := grpc.Dial(address, grpc.WithInsecure())

这种模式下，client 和 server 都不会进行通信认证，其实是不安全的。下面我们来看看安全模式下应该如何通信

#### server

	package main
	
	import (
		"context"
		"log"
		"net"
	
		"google.golang.org/grpc"
		"google.golang.org/grpc/credentials"
	
		pb "google.golang.org/grpc/examples/helloworld/helloworld"
	)
	
	const (
		port = ":50051"
	)
	
	// server is used to implement helloworld.GreeterServer.
	type server struct{}
	
	// SayHello implements helloworld.GreeterServer
	func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
		log.Printf("Received: %v", in.Name)
		return &pb.HelloReply{Message: "Hello " + in.Name}, nil
	}
	
	func main() {
		c, err := credentials.NewServerTLSFromFile("../keys/server.pem", "../keys/server.key")
	    	if err != nil {
	        	log.Fatalf("credentials.NewServerTLSFromFile err: %v", err)
	    	}
	
		lis, err := net.Listen("tcp", port)
		if err != nil {
			log.Fatalf("failed to listen: %v", err)
		}
		s := grpc.NewServer(grpc.Creds(c))
		pb.RegisterGreeterServer(s, &server{})
		if err := s.Serve(lis); err != nil {
			log.Fatalf("failed to serve: %v", err)
		}
	}

#### client
	package main
	
	import (
		"context"
		"log"
		"os"
		"time"
	
		"google.golang.org/grpc"
		"google.golang.org/grpc/credentials"
	
		pb "helloauth/helloworld"
	)
	
	const (
		address     = "localhost:50051"
		defaultName = "world"
	)
	
	func main() {
		cred, err := credentials.NewClientTLSFromFile("../keys/server.pem", "helloauth")
	    	if err != nil {
	        	log.Fatalf("credentials.NewClientTLSFromFile err: %v", err)
	    	}
	
		// Set up a connection to the server.
		conn, err := grpc.Dial(address, grpc.WithTransportCredentials(cred))
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

这里的代码已经上传 github 了，详见：https://github.com/diubrother/helloauth

### 4、grpc 认证鉴权源码解读
#### server
先来看 server 端，server 端根据 server 的公钥和私钥生成了一个 TransportCredentials ，如下：

	c, err := credentials.NewServerTLSFromFile("../keys/server.pem", "../keys/server.key")

	func NewServerTLSFromFile(certFile, keyFile string) (TransportCredentials, error) {
		cert, err := tls.LoadX509KeyPair(certFile, keyFile)
		if err != nil {
			return nil, err
		}
		return NewTLS(&tls.Config{Certificates: []tls.Certificate{cert}}), nil
	}

看一下 NewTLS 这个方法，他其实就返回了一个 tlsCreds 的结构体，这个结构体实现了 TransportCredentials 这个接口，包括 ClientHandshake 和 ServerHandshake 。

	func NewTLS(c *tls.Config) TransportCredentials {
		tc := &tlsCreds{cloneTLSConfig(c)}
		tc.config.NextProtos = appendH2ToNextProtos(tc.config.NextProtos)
		return tc
	}

来看一下服务端握手的方法 ServerHandshake，可以发现其底层还是调用 go 的 tls 包去实现 tls 认证鉴权。

	func (c *tlsCreds) ServerHandshake(rawConn net.Conn) (net.Conn, AuthInfo, error) {
		conn := tls.Server(rawConn, c.config)
		if err := conn.Handshake(); err != nil {
			return nil, nil, err
		}
		return internal.WrapSyscallConn(rawConn, conn), TLSInfo{conn.ConnectionState()}, nil
	}

#### client
和 server 端类似，client 端也是通过公钥和服务名先创建一个 TransportCredentials

	cred, err := credentials.NewClientTLSFromFile("../keys/server.pem", "helloauth")

看一下 NewClientTLSFromFile 这个方法，发现它也是调用了相同的 NewTLS 方法返回了一个 tlsCreds 结构体，跟 server 简直一模一样。

	func NewTLS(c *tls.Config) TransportCredentials {
		tc := &tlsCreds{cloneTLSConfig(c)}
		tc.config.NextProtos = appendH2ToNextProtos(tc.config.NextProtos)
		return tc
	}

接下来在创建客户端连接时，将 tlsCreds 这个结构体传了进去。

	conn, err := grpc.Dial(address, grpc.WithTransportCredentials(cred))

Dial —— > DialContext 方法中有这么一段代码，将我们传入的 serverName 也就是 “helloauth" 赋值给了 clientConn 的 authority 这个字段。

		creds := cc.dopts.copts.TransportCredentials
		if creds != nil && creds.Info().ServerName != "" {
			cc.authority = creds.Info().ServerName
		} else if cc.dopts.insecure && cc.dopts.authority != "" {
			cc.authority = cc.dopts.authority
		} else {
			// Use endpoint from "scheme://authority/endpoint" as the default
			// authority for ClientConn.
			cc.authority = cc.parsedTarget.Endpoint
		}

#### 认证过程
#### client
那什么时候开始认证呢？先来说说 client。

client 的认证其实是在调用 connect 方法的时候，在之前讲述负载均衡时降到了，在 acBalancerWrapper 里面有一个 UpdateAddresses 方法，调用 ac.connect() ——> ac.resetTransport() ——> ac.tryAllAddrs ——> ac.createTransport ——> transport.NewClientTransport ——> newHTTP2Client 方法时，有这么一段代码：

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
		if transportCreds != nil {
			scheme = "https"
			conn, authInfo, err = transportCreds.ClientHandshake(connectCtx, addr.Authority, conn)
			if err != nil {
				return nil, connectionErrorf(isTemporary(err), err, "transport: authentication handshake failed: %v", err)
			}
			isSecure = true
		}

这里即调用了tlsCreds 的 ClientHandshake 方法进行握手，实现客户端的认证。

#### server 
再来说说 server

server 的认证其实是在调用 Serve ——> handleRawConn ——> useTransportAuthenticator 方法，调用了 s.opts.creds.ServerHandshake(rawConn) 方法，其底层也是调用 tlsCreds ServerHandshake 方法进行服务端握手。

	func (s *Server) useTransportAuthenticator(rawConn net.Conn) (net.Conn, credentials.AuthInfo, error) {
		if s.opts.creds == nil {
			return rawConn, nil, nil
		}
		return s.opts.creds.ServerHandshake(rawConn)
	}
