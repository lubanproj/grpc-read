
### grpc quick start

我们分析 go 版本的 grpc 实现，所以这里主要讲解 grpc-go 的安装和使用

#### 1、安装

go 语言版本的 grpc 安装需要 1.6 以上的 go 版本，所以你需要先执行 go version 查看 go  版本，假如版本低于 1.6 则需要先升级。

官网上给的安装方式是 

	go get -u google.golang.org/grpc

因为 google.golang.org 这个域名在国内会被墙，所以假如可以翻墙的计算机可以用这种方式，不能翻墙的可以通过以下方式安装

	git clone https://github.com/grpc/grpc-go.git  $GOPATH/src/google.golang.org/grpc
	git clone https://github.com/golang/net.git  $GOPATH/src/golang.org/x/net
	git clone https://github.com/golang/text.git $GOPATH/src/golang.org/x/text 
	go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
	git clone https://github.com/google/go-genproto.git $GOPATH/src/google.golang.org/genproto

	 cd $GOPATH/src/
	 go install google.golang.org/grpc

假如安装不成功则说明你 GOPATH 没有配置，可以先配置 GOPATH 后再执行
		
	// 配置 GOPATH
	export GOPATH=/Users/delvin/go

#### 2、hello world

   grpc-go 官方提供了一些 examples ，都放在 examples 目录下，examples 目录下有三个目录，features 目录主要是 grpc 的一些特写使用，包括路由寻址、keep-alive、负载均衡等。helloworld 目录下主要是提供了一个 helloworld demo。route_guide 目录主要提供了对 grpc 四种调用方式：Simple RPC、Client-side streaming RPC、Server-side streaming RPC、Bidirectional streaming RPC 的模拟调用 demo。

   我们通过 grpc 提供的 helloworld demo 来简单了解下如何使用 grpc 。

   之前我们说到了 grpc 是通过 protocol buffer 来实现接口的定义的。helloworld 包下已经给我们定义好了一个 helloworld.proto 如下：
	
	syntax = "proto3";
	
	option java_multiple_files = true;
	option java_package = "io.grpc.examples.helloworld";
	option java_outer_classname = "HelloWorldProto";
	
	package helloworld;
	
	// The greeting service definition.
	service Greeter {
	  // Sends a greeting
	  rpc SayHello (HelloRequest) returns (HelloReply) {}
	}
	
	// The request message containing the user's name.
	message HelloRequest {
	  string name = 1;
	}
	
	// The response message containing the greetings
	message HelloReply {
	  string message = 1;
	}

可以看到 SayHello 这个结构体里面已经定义好了一个 rpc Service 调用，
	
	 rpc SayHello (HelloRequest) returns (HelloReply) {}


在 Service 中包含一个方法，SayHello, 支持传入一个 HelloRequest 的参数，返回一个 HelloReply 的响应。这两个结构体分别也在 proto 文件里面定义了。

通过 protoc 可以生成一个 pb.go 文件（这里 demo ）里面已经生成好了。然后编写一个 client 和 server 即可实现一个完整的 rpc 调用链路，这里 demo 里面也提供了，我们先直接运行一下：

	cd  $GOPATH/src/google.golang.org/grpc/examples/helloworld

先在一个 terminal 下执行：
	
	go run greeter_server/main.go

然后在另一个 terminal 下执行：

	go run greeter_client/main.go

此时 client 会输出 Greeting:  Helloworld

	2019/08/03 15:57:46 Greeting: Helloworld


#### 3、自定义一个 service

  这里也是 grpc docs 上一个相同的例子，我贴上来介绍一下，为了让本节内容更完整。
	
（1）在 pb 文件里面添加一个 rpc 方法

		// The greeting service definition.
		service Greeter {
		  // Sends a greeting
		  rpc SayHello (HelloRequest) returns (HelloReply) {}
		  // Sends another greeting
		  rpc SayHelloAgain (HelloRequest) returns (HelloReply) {}
		}
		
		// The request message containing the user's name.
		message HelloRequest {
		  string name = 1;
		}
		
		// The response message containing the greetings
		message HelloReply {
		  string message = 1;
		}

（2）生成 pb.go 文件

	protoc -I helloworld/ helloworld/helloworld.proto --go_out=plugins=grpc:helloworld

（3）在 greeter_server/main.go 下添加一个方法，入参和返回值和 SayHello 完全相同，使用已经定义过的结构体 HelloRequest 和 HelloReply

	func (s *server) SayHelloAgain(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	        return &pb.HelloReply{Message: "Hello again " + in.Name}, nil
	}

（4）在 greeter_client/main.go 下添加一个方法调用，调用 SayHelloAgain 这个方法

	r, err = c.SayHelloAgain(ctx, &pb.HelloRequest{Name: name})
	if err != nil {
	        log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.Message)

（5）执行

先在一个 terminal 下执行：
	
	go run greeter_server/main.go

然后在另一个 terminal 下执行：

	go run greeter_client/main.go

可以看到 client 的 terminal 输出如下：

	DEVLINTANG-MB0:helloworld delvin$ go run greeter_client/main.go
	2019/08/03 16:18:24 Greeting: Hello world
	2019/08/03 16:18:24 Greeting: Hello again world

这篇内容主要介绍了 grpc 的安装以及基本使用。分析了通过 grpc 输出 hello world 的基本要素和过程。
