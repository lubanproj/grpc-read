### 协议编解码器

一般的协议都会包括协议头和协议体，对于业务而言，一般只关心需要发送的业务数据。所以，协议头的内容一般是框架自动帮忙填充。将业务数据包装成指定协议格式的数据包就是编码的过程，从指定协议格式中的数据包中取出业务数据的过程就是解码的过程。

每个 rpc 框架基本都有自己的编解码器，下面我们就来说说 grpc 的编解码过程。

### grpc 解码
我们还是从我们的 examples 目录下的 helloworld demo 中 server 的 main 函数入手

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

在 s.Serve(lis) ——> s.handleRawConn(rawConn) —— > s.serveStreams(st) ——> s.handleStream(st, stream, s.traceInfo(st, stream)) ——> s.processUnaryRPC(t, stream, srv, md, trInfo) 方法中有一段代码：
	
	sh := s.opts.statsHandler
	...
	df := func(v interface{}) error {
			if err := s.getCodec(stream.ContentSubtype()).Unmarshal(d, v); err != nil {
				return status.Errorf(codes.Internal, "grpc: error unmarshalling request: %v", err)
			}
			if sh != nil {
				sh.HandleRPC(stream.Context(), &stats.InPayload{
					RecvTime:   time.Now(),
					Payload:    v,
					WireLength: payInfo.wireLength,
					Data:       d,
					Length:     len(d),
				})
			}
			if binlog != nil {
				binlog.Log(&binarylog.ClientMessage{
					Message: d,
				})
			}
			if trInfo != nil {
				trInfo.tr.LazyLog(&payload{sent: false, msg: v}, true)
			}
			return nil
	}

这段代码的逻辑先调 getCodec 获取解包类，然后调用这个类的 Unmarshal 方法进行解包。将业务数据取出来，然后调用 handler 进行处理。

	func (s *Server) getCodec(contentSubtype string) baseCodec {
		if s.opts.codec != nil {
			return s.opts.codec
		}
		if contentSubtype == "" {
			return encoding.GetCodec(proto.Name)
		}
		codec := encoding.GetCodec(contentSubtype)
		if codec == nil {
			return encoding.GetCodec(proto.Name)
		}
		return codec
	}

我们来看 getCodec 这个方法，它是通过 contentSubtype 这个字段来获取解包类的。假如不设置 contentSubtype ，那么默认会用名字为 proto 的解码器。

我们来看看 contentSubtype 是如何设置的。之前说到了 grpc 的底层默认是基于 http2 的。在 serveHttp 时调用了 NewServerHandlerTransport 这个方法来创建一个 ServerTransport，然后我们发现，其实就是根据 content-type 这个字段去生成的。

	func NewServerHandlerTransport(w http.ResponseWriter, r *http.Request, stats stats.Handler) (ServerTransport, error) {
		...
	
		contentType := r.Header.Get("Content-Type")
		// TODO: do we assume contentType is lowercase? we did before
		contentSubtype, validContentType := contentSubtype(contentType)
		if !validContentType {
			return nil, errors.New("invalid gRPC request content-type")
		}
		if _, ok := w.(http.Flusher); !ok {
			return nil, errors.New("gRPC requires a ResponseWriter supporting http.Flusher")
		}
	
		st := &serverHandlerTransport{
			rw:             w,
			req:            r,
			closedCh:       make(chan struct{}),
			writes:         make(chan func()),
			contentType:    contentType,
			contentSubtype: contentSubtype,
			stats:          stats,
		}
	}

我们来看看 contentSubtype 这个方法 。

	...
	baseContentType = "application/grpc"
	...
	func contentSubtype(contentType string) (string, bool) {
		if contentType == baseContentType {
			return "", true
		}
		if !strings.HasPrefix(contentType, baseContentType) {
			return "", false
		}
		// guaranteed since != baseContentType and has baseContentType prefix
		switch contentType[len(baseContentType)] {
		case '+', ';':
			// this will return true for "application/grpc+" or "application/grpc;"
			// which the previous validContentType function tested to be valid, so we
			// just say that no content-subtype is specified in this case
			return contentType[len(baseContentType)+1:], true
		default:
			return "", false
		}
	}

可以看到 grpc 协议默认以 application/grpc 开头，假如不一这个开头会返回错误，假如我们想使用 json 的解码器，应该设置 content-type = application/grpc+json 。下面是一个基于 grpc 协议的请求 request ：

	HEADERS (flags = END_HEADERS)
	:method = POST
	:scheme = http
	:path = /google.pubsub.v2.PublisherService/CreateTopic
	:authority = pubsub.googleapis.com
	grpc-timeout = 1S
	content-type = application/grpc+proto
	grpc-encoding = gzip
	authorization = Bearer y235.wef315yfh138vh31hv93hv8h3v
	
	DATA (flags = END_STREAM)
	<Length-Prefixed Message>

详细可参考  [proto-http2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)

怎么拿的呢，再看一下 encoding.getCodec 方法

	func GetCodec(contentSubtype string) Codec {
		return registeredCodecs[contentSubtype]
	}

它其实取得是 registeredCodecs 这个 map 中的 codec，这个 map 是 RegisterCodec 方法注册进去的。

	var registeredCodecs = make(map[string]Codec)
	
	func RegisterCodec(codec Codec) {
		if codec == nil {
			panic("cannot register a nil Codec")
		}
		if codec.Name() == "" {
			panic("cannot register Codec with empty string result for Name()")
		}
		contentSubtype := strings.ToLower(codec.Name())
		registeredCodecs[contentSubtype] = codec
	}

毫无疑问， encoding 目录的 proto 包下肯定在初始化时调用注册方法了。果然

	func init() {
		encoding.RegisterCodec(codec{})
	}

绕了一圈，调用的其实是 proto 的 Unmarshal 方法，如下：

	func (codec) Unmarshal(data []byte, v interface{}) error {
		protoMsg := v.(proto.Message)
		protoMsg.Reset()
	
		if pu, ok := protoMsg.(proto.Unmarshaler); ok {
			// object can unmarshal itself, no need for buffer
			return pu.Unmarshal(data)
		}
	
		cb := protoBufferPool.Get().(*cachedProtoBuffer)
		cb.SetBuf(data)
		err := cb.Unmarshal(protoMsg)
		cb.SetBuf(nil)
		protoBufferPool.Put(cb)
		return err
	}

### grpc 编码
在剖析解码代码的基础上，编码代码就很轻松了，其实直接找到 encoding 目录的 proto 包，看 Marshal 方法在哪儿被调用就行了。

于是我们很快就找到了调用路径，也是这个路径：

 s.Serve(lis) ——> s.handleRawConn(rawConn) —— > s.serveStreams(st) ——> s.handleStream(st, stream, s.traceInfo(st, stream)) ——> s.processUnaryRPC(t, stream, srv, md, trInfo) 

processUnaryRPC 方法中有一段 server 发送响应数据的代码。其实也就是这一行：

	if err := s.sendResponse(t, stream, reply, cp, opts, comp); err != nil {
	
我们其实也能猜到，发送数据给 client 之前肯定要编码。果然调用了 encode 方法 		

	func (s *Server) sendResponse(t transport.ServerTransport, stream *transport.Stream, msg interface{}, cp Compressor, opts *transport.Options, comp encoding.Compressor) error {
		data, err := encode(s.getCodec(stream.ContentSubtype()), msg)
		if err != nil {
			grpclog.Errorln("grpc: server failed to encode response: ", err)
			return err
		}
		...
	}

来看一下 encode

	func encode(c baseCodec, msg interface{}) ([]byte, error) {
		if msg == nil { // NOTE: typed nils will not be caught by this check
			return nil, nil
		}
		b, err := c.Marshal(msg)
		if err != nil {
			return nil, status.Errorf(codes.Internal, "grpc: error while marshaling: %v", err.Error())
		}
		if uint(len(b)) > math.MaxUint32 {
			return nil, status.Errorf(codes.ResourceExhausted, "grpc: message too large (%d bytes)", len(b))
		}
		return b, nil
	}

它调用了 c.Marshal 方法， Marshal 方法其实是 baseCodec 定义的一个通用抽象方法

	type baseCodec interface {
		Marshal(v interface{}) ([]byte, error)
		Unmarshal(data []byte, v interface{}) error
	}

proto 实现了 baseCodec，前面说到了通过 s.getCodec(stream.ContentSubtype(),msg) 获取到的其实是 contentType 里面设置的协议名称，不设置的话默认取 proto 的编码器。所以最终是调用了 proto 包下的 Marshal 方法，如下：

	func (codec) Marshal(v interface{}) ([]byte, error) {
		if pm, ok := v.(proto.Marshaler); ok {
			// object can marshal itself, no need for buffer
			return pm.Marshal()
		}
	
		cb := protoBufferPool.Get().(*cachedProtoBuffer)
		out, err := marshal(v, cb)
	
		// put back buffer and lose the ref to the slice
		cb.SetBuf(nil)
		protoBufferPool.Put(cb)
		return out, err
	}

ok，那么至此，grpc 的整个编解码的流程我们就已经剖析完了
