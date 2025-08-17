> 本文原发表于：https://diu.life/lessons/grpc-read/grpc-interceptor-implementation/
> 最新版本请访问原文链接

## grpc 源码解读 —— 从 0 到 1 实现拦截器

拦截器，通俗点说，就是在执行一段代码之前或者之后，去执行另外一段代码。
拦截器在业界知名框架中的运用非常普遍。包括 Spring 、Grpc 等框架中都有拦截器的实现。接下来我们想办法从 0 到 1 自己实现一个拦截器。以下的实现主要使用 go 语言讲解。

假设有一个方法 handler(ctx context.Context) ，我想要给这个方法赋予一个能力：允许在这个方法执行之前能够打印一行日志。

#### 1、定义结构

于是我们轻而易举得想到了定义一个结构 interceptor 这个结构包含两个参数，一个 context 和 一个 handler
		
		type interceptor func(ctx context.Context, handler func(ctx context.Context) )

为了能够更加方便，我们将 handler 单独定义成一种类型：

		type interceptor func(ctx context.Context, h handler)

		type handler func(ctx context.Context)


#### 2、申明赋值

接下来，为了实现我们的目标，对 handler 的每个操作，我们都需要先经过 interceptor 。于是我们申明两个 interceptor 和 handler 的变量并赋值
		
		var h = func(ctx context.Context) {
				fmt.Println("do something ...")
		}
		
		var inter1 = func(ctx context.Context, h handler) {
				fmt.Println("interceptor1")
				h(ctx)
		}

#### 3、编写执行函数
编写一个执行函数，看看效果

		func main() {
		
			var ctx context.Context
		
			var ceps []interceptor
		
			var h = func(ctx context.Context) {
				fmt.Println("do something ...")
			}
		
			var inter1 = func(ctx context.Context, h handler) {
				fmt.Println("interceptor1")
				h(ctx)
			}
		
			ceps = append(ceps, inter1)
		
			for _ , cep := range ceps {
				cep(ctx, h)
			}
		
		}

输出结果为 ：

		interceptor1
		do something ...

ok，我们已经完成了实现这个方法之前 输出一行内容。

是不是大功告成了呢？ wait ... 我们再来加一个 interceptor 试试，于是我们又加了一个 interceptor

		   var inter2 = func(ctx context.Context, h handler) {
				fmt.Println("interceptor2")
				h(ctx)
			}

同样，我们编写一个执行函数

		func main() {
		
			var ctx context.Context
		
			var ceps []interceptor
		
			var h = func(ctx context.Context) {
				fmt.Println("do something ...")
			}
		
			var inter1 = func(ctx context.Context, h handler) {
				fmt.Println("interceptor1")
				h(ctx)
			}
			var inter2 = func(ctx context.Context, h handler) {
				fmt.Println("interceptor2")
				h(ctx)
			}
		
			ceps = append(ceps, inter1, inter2)
		
			for _ , cep := range ceps {
				cep(ctx, h)
			}
		
		}

执行结果如下：

		interceptor1
		do something ...
		interceptor2
		do something ...

可以看到，在 handler 之前确实输出了两行内容。但是总感觉哪里不太对？？？ wait ... handler 竟然执行了两次。这可不是我们想要的效果，我们希望无论打印多少行内容，应该保证 handler 只执行一次。

#### 4、借鉴 grpc-go 

于是我们开始想办法，怎么才能让 handler 只执行一次呢？ 想啊想，想了一会儿没想到，这个时候灵光一闪，可以借助前人的智慧啊...... grpc 中肯定有实现，我们先来 “借鉴” 一下，毕竟他山之石，可以攻玉嘛.....

翻开 grpc-go 的源码，直接找到 helloworld demo client 端的 main 函数，grpc.Dial ——> DialContext ，里面有一行

		chainUnaryClientInterceptors(cc)
 
 别问我是怎么找到的，直接去看我之前关于 client 的源码解读 [grpc quick start](https://github.com/lubanproj/grpc_read/blob/master/4-grpc%20hello%20world%20client%20%E8%A7%A3%E6%9E%90.md) 就知道了（这广告貌似一点也不违合 hhh）。

 来看看这个函数，这个函数就有点牛逼了。

		// chainUnaryClientInterceptors chains all unary client interceptors into one.
		func chainUnaryClientInterceptors(cc *ClientConn) {
			interceptors := cc.dopts.chainUnaryInts
			// Prepend dopts.unaryInt to the chaining interceptors if it exists, since unaryInt will
			// be executed before any other chained interceptors.
			if cc.dopts.unaryInt != nil {
				interceptors = append([]UnaryClientInterceptor{cc.dopts.unaryInt}, interceptors...)
			}
			var chainedInt UnaryClientInterceptor
			if len(interceptors) == 0 {
				chainedInt = nil
			} else if len(interceptors) == 1 {
				chainedInt = interceptors[0]
			} else {
				chainedInt = func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error {
					return interceptors[0](ctx, method, req, reply, cc, getChainUnaryInvoker(interceptors, 0, invoker), opts...)
				}
			}
			cc.dopts.unaryInt = chainedInt
		}

chains all unary client interceptors into one. 这句话告诉我们，这个函数把所有拦截器串成了一个拦截器。这是怎么实现的呢？这不就是我们上面碰到的问题吗！来瞅瞅~~~

		// getChainUnaryInvoker recursively generate the chained unary invoker.
		func getChainUnaryInvoker(interceptors []UnaryClientInterceptor, curr int, finalInvoker UnaryInvoker) UnaryInvoker {
			if curr == len(interceptors)-1 {
				return finalInvoker
			}
			return func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error {
				return interceptors[curr+1](ctx, method, req, reply, cc, getChainUnaryInvoker(interceptors, curr+1, finalInvoker), opts...)
			}
		}

原来是通过 getChainUnaryInvoker 这个方法，返回一个 UnaryInvoker ，这一个 UnaryInvoker 也是一个函数

	type UnaryInvoker func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error

在这个 UnaryInvoker 实例化时会去调用第 curr+1 个 interceptors。也就是最终会返回这样一个结构：

 ![请求流程](https://img-blog.csdnimg.cn/20190827163339853.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpdWJyb3RoZXI=,size_16,color_FFFFFF,t_70)
接下来将这个结构赋值给了 cc.dopts.unaryInt ，但此时并没有调用。什么时候开始调用的呢？

还记得我们之前 client SayHello 的时候调用的这一行代码吗？

		err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)

在 Invoke 这个函数里面进行了真正的调用

		func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
			// allow interceptor to see all applicable call options, which means those
			// configured as defaults from dial option as well as per-call options
			opts = combine(cc.dopts.callOptions, opts)
		
			if cc.dopts.unaryInt != nil {
				return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
			}
			return invoke(ctx, method, args, reply, cc, opts...)
		}

这一行代码 cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...) 才是入口，就如同多米诺骨牌一样，只要这里开始调用，然后就会一直层层调用下去，直到所有的 interceptor 调用完成。

这里的代码虽然不多，但其实设计得非常精妙。包括每一个结构的定义。前辈果然还是前辈

#### 5、从 0 到 1 实现一个拦截器

仔细研究完 grpc 的经典实现，这里我们就可以自己来实现一个简化版的拦截器了。

##### 5.1 重新定义结构
 
首先，之前我们的疑问是如何让 handler 只执行一遍。这里我们将原来的 handler 升级一下，成为 Invoker , 重新定义一个 handler ，用于在 Invoker  执行之前处理某些事情。

	type invoker func(ctx context.Context, interceptors []interceptor2 , h handler) error
	type handler func(ctx context.Context)

之前的 interceptor 也需要更改一下，需要传入我们的 invoker 和 handler

	type interceptor2 func(ctx context.Context, h handler, ivk invoker) error

##### 5.2 串联所有 interceptor

接下来，我们需要把所有的 interceptor 串联起来

	func getInvoker(ctx context.Context, interceptors []interceptor2 , cur int, ivk invoker) invoker{
		 if cur == len(interceptors) - 1 {
			return ivk
		}
		 return func(ctx context.Context, interceptors []interceptor2 , h handler) error{
			return 	interceptors[cur+1](ctx, h, getInvoker(ctx,interceptors, cur+1, ivk))
		}
	}

##### 5.3 返回第一个 interceptor 作为入口

		func getChainInterceptor(ctx context.Context, interceptors []interceptor2 , ivk invoker) interceptor2 {
			if len(interceptors) == 0 {
				return nil
			}
			if len(interceptors) == 1 {
				return interceptors[0]
			}
			return func(ctx context.Context, h handler, ivk invoker) error {
				return interceptors[0](ctx, h, getInvoker(ctx, interceptors, 0, ivk))
			}
		} 

##### 5.4  编写执行用例

完整的执行用例如下：

	package main
	
	import (
		"context"
		"fmt"
	)
	
	type interceptor2 func(ctx context.Context, h handler, ivk invoker) error
	
	type handler func(ctx context.Context)
	
	type invoker func(ctx context.Context, interceptors []interceptor2 , h handler) error
	
	func main() {
	
		var ctx context.Context
		var ceps []interceptor2
		var h = func(ctx context.Context) {
			fmt.Println("do something")
		}
	
		var inter1 = func(ctx context.Context, h handler, ivk invoker) error{
			h(ctx)
			return ivk(ctx,ceps,h)
		}
		var inter2 = func(ctx context.Context, h handler, ivk invoker) error{
			h(ctx)
			return ivk(ctx,ceps,h)
		}
	
		var inter3 = func(ctx context.Context, h handler, ivk invoker) error{
			h(ctx)
			return 	ivk(ctx,ceps,h)
		}
	
		ceps = append(ceps, inter1, inter2, inter3)
		var ivk = func(ctx context.Context, interceptors []interceptor2 , h handler) error {
			fmt.Println("invoker start")
			return nil
		}
	
		cep := getChainInterceptor(ctx, ceps,ivk)
		cep(ctx, h,ivk)
	
	}
	
	func getChainInterceptor(ctx context.Context, interceptors []interceptor2 , ivk invoker) interceptor2 {
		if len(interceptors) == 0 {
			return nil
		}
		if len(interceptors) == 1 {
			return interceptors[0]
		}
		return func(ctx context.Context, h handler, ivk invoker) error {
			return interceptors[0](ctx, h, getInvoker(ctx, interceptors, 0, ivk))
		}
	
	}
	
	
	func getInvoker(ctx context.Context, interceptors []interceptor2 , cur int, ivk invoker) invoker{
		 if cur == len(interceptors) - 1 {
			return ivk
		}
		 return func(ctx context.Context, interceptors []interceptor2 , h handler) error{
			return 	interceptors[cur+1](ctx, h, getInvoker(ctx,interceptors, cur+1, ivk))
		}
	}


##### 5.5 执行结果

		do something
		do something
		do something
		invoker start

执行结果如上，可以看到每次 Invoker 执行前我们都调用了 handler，但是 Invoker 只被调用了一次，完美地实现了我们的诉求，一个简化版的拦截器诞生了。

