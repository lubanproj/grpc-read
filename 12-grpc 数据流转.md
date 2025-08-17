> 本文原发表于：https://diu.life/lessons/grpc-read/grpc-data-flow/
> 最新版本请访问原文链接

#  grpc 数据流转

阅读本文的前提是你对 grpc 协议的编解码和 协议打解包过程都比较清楚了，假如不是很了解可以先去阅读 [《10 - grpc 协议编解码器》](https://github.com/lubanproj/grpc_read/blob/master/10-grpc%20%E5%8D%8F%E8%AE%AE%E7%BC%96%E8%A7%A3%E7%A0%81%E5%99%A8.md) 和 [《11 - grpc 协议解包过程全剖析》](https://github.com/lubanproj/grpc_read/blob/master/11-grpc%20%E5%8D%8F%E8%AE%AE%E8%A7%A3%E5%8C%85%E8%BF%87%E7%A8%8B%E5%85%A8%E5%89%96%E6%9E%90.md)

## 再谈协议
我们知道协议是一款 rpc 框架的基础。协议里面定义了一次客户端需要携带的信息，包括请求的后端服务名 ServiceName，方法名 Method、超时时间 Timeout、编码 Encoding、认证信息 Authority 等等。

前面我们已经说到了，grpc 是基于 http2 协议的，我们来看看 grpc 协议里面的一些关键信息：

![grpc 协议](https://images.xiaozhuanlan.com/photo/2020/211eb35fa812fda032a342bc60611cb9.png)

可以看到，一次请求需要携带这么多信息，server 会根据 client 携带的这些信息来进行相应的处理。那么这些协议里面定义的内容要如何被传递下去呢？

## 数据承载体
为了回答上面的问题，我们需要一个数据承载体结构，来保存协议里面的一些需要透传的一些重要信息，比如 Method 等。在 grpc 中，这个结构就是 Stream， 我们来看一下 Stream 的定义。

	// Stream represents an RPC in the transport layer.
	type Stream struct {
		id           uint32
		st           ServerTransport    // nil for client side Stream
		ctx          context.Context    // the associated context of the stream
		cancel       context.CancelFunc // always nil for client side Stream
		done         chan struct{}      // closed at the end of stream to unblock writers. On the client side.
		ctxDone      <-chan struct{}    // same as done chan but for server side. Cache of ctx.Done() (for performance)
		method       string             // the associated RPC method of the stream
		recvCompress string
		sendCompress string
		buf          *recvBuffer
		trReader     io.Reader
		fc           *inFlow
		wq           *writeQuota
	
		// Callback to state application's intentions to read data. This
		// is used to adjust flow control, if needed.
		requestRead func(int)
	
		headerChan       chan struct{} // closed to indicate the end of header metadata.
		headerChanClosed uint32        // set when headerChan is closed. Used to avoid closing headerChan multiple times.
	
		// hdrMu protects header and trailer metadata on the server-side.
		hdrMu sync.Mutex
		// On client side, header keeps the received header metadata.
		//
		// On server side, header keeps the header set by SetHeader(). The complete
		// header will merged into this after t.WriteHeader() is called.
		header  metadata.MD
		trailer metadata.MD // the key-value map of trailer metadata.
	
		noHeaders bool // set if the client never received headers (set only after the stream is done).
	
		// On the server-side, headerSent is atomically set to 1 when the headers are sent out.
		headerSent uint32
	
		state streamState
	
		// On client-side it is the status error received from the server.
		// On server-side it is unused.
		status *status.Status
	
		bytesReceived uint32 // indicates whether any bytes have been received on this stream
		unprocessed   uint32 // set if the server sends a refused stream or GOAWAY including this stream
	
		// contentSubtype is the content-subtype for requests.
		// this must be lowercase or the behavior is undefined.
		contentSubtype string
	}

### server 端 Stream 的构造
接下来我们来看看 server 端 Stream 的构造。前面的内容已经说过 server 的处理流程了。我们直接进入 serveStreams 这个方法。路径为：s.Serve(lis) ——> s.handleRawConn(rawConn) ——> s.serveStreams(st) 

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

最上层 HandleStreams 是对 http2 数据帧的处理。grpc 一共处理了 MetaHeadersFrame 、DataFrame、RSTStreamFrame、SettingsFrame、PingFrame、WindowUpdateFrame、GoAwayFrame 等 7 种帧。

	// HandleStreams receives incoming streams using the given handler. This is
	// typically run in a separate goroutine.
	// traceCtx attaches trace to ctx and returns the new context.
	func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
		defer close(t.readerDone)
		for {
			frame, err := t.framer.fr.ReadFrame()
			atomic.StoreUint32(&t.activity, 1)
			if err != nil {
				if se, ok := err.(http2.StreamError); ok {
					warningf("transport: http2Server.HandleStreams encountered http2.StreamError: %v", se)
					t.mu.Lock()
					s := t.activeStreams[se.StreamID]
					t.mu.Unlock()
					if s != nil {
						t.closeStream(s, true, se.Code, false)
					} else {
						t.controlBuf.put(&cleanupStream{
							streamID: se.StreamID,
							rst:      true,
							rstCode:  se.Code,
							onWrite:  func() {},
						})
					}
					continue
				}
				if err == io.EOF || err == io.ErrUnexpectedEOF {
					t.Close()
					return
				}
				warningf("transport: http2Server.HandleStreams failed to read frame: %v", err)
				t.Close()
				return
			}
			switch frame := frame.(type) {
			case *http2.MetaHeadersFrame:
				if t.operateHeaders(frame, handle, traceCtx) {
					t.Close()
					break
				}
			case *http2.DataFrame:
				t.handleData(frame)
			case *http2.RSTStreamFrame:
				t.handleRSTStream(frame)
			case *http2.SettingsFrame:
				t.handleSettings(frame)
			case *http2.PingFrame:
				t.handlePing(frame)
			case *http2.WindowUpdateFrame:
				t.handleWindowUpdate(frame)
			case *http2.GoAwayFrame:
				// TODO: Handle GoAway from the client appropriately.
			default:
				errorf("transport: http2Server.HandleStreams found unhandled frame type %v.", frame)
			}
		}
	}

对于每一次请求而言，client 一定会先发 HeadersFrame 这个帧，grpc 这里是直接使用 http2 工具包进行实现，直接处理的 MetaHeadersFrame 帧，这个帧的定义为：

	// A MetaHeadersFrame is the representation of one HEADERS frame and
	// zero or more contiguous CONTINUATION frames and the decoding of
	// their HPACK-encoded contents.
	//
	// This type of frame does not appear on the wire and is only returned
	// by the Framer when Framer.ReadMetaHeaders is set.
	type MetaHeadersFrame struct {
	        *HeadersFrame
	        Fields []hpack.HeaderField
	        Truncated bool
	}

所以是在 MetaHeadersFrame 这个帧里去处理包头数据。所以会去执行 operateHeaders 这个方法，在这个方法里面会去构造一个 stream ，这个 stream 里面包含了传输层请求上下文的数据。包括方法名等。

	s := &Stream{
			id:             streamID,
			st:             t,
			buf:            buf,
			fc:             &inFlow{limit: uint32(t.initialWindowSize)},
			recvCompress:   state.data.encoding,
			method:         state.data.method,
			contentSubtype: state.data.contentSubtype,
		}

构造完 stream 后，接下来 tranport 对数据的处理都会将 stream 层层透传下去。所以整个请求内所需要的数据都从 stream 中可以得到，这样就实现了 server 端的数据流转。

### client 端数据流转

与 server 相对应，client 端也有一个 clientStream 结构，定义如下：

	// clientStream implements a client side Stream.
	type clientStream struct {
		callHdr  *transport.CallHdr
		opts     []CallOption
		callInfo *callInfo
		cc       *ClientConn
		desc     *StreamDesc
	
		codec baseCodec
		cp    Compressor
		comp  encoding.Compressor
	
		cancel context.CancelFunc // cancels all attempts
	
		sentLast  bool // sent an end stream
		beginTime time.Time
	
		methodConfig *MethodConfig
	
		ctx context.Context // the application's context, wrapped by stats/tracing
	
		retryThrottler *retryThrottler // The throttler active when the RPC began.
	
		binlog *binarylog.MethodLogger // Binary logger, can be nil.
		// serverHeaderBinlogged is a boolean for whether server header has been
		// logged. Server header will be logged when the first time one of those
		// happens: stream.Header(), stream.Recv().
		//
		// It's only read and used by Recv() and Header(), so it doesn't need to be
		// synchronized.
		serverHeaderBinlogged bool
	
		mu                      sync.Mutex
		firstAttempt            bool       // if true, transparent retry is valid
		numRetries              int        // exclusive of transparent retry attempt(s)
		numRetriesSincePushback int        // retries since pushback; to reset backoff
		finished                bool       // TODO: replace with atomic cmpxchg or sync.Once?
		attempt                 *csAttempt // the active client stream attempt
		// TODO(hedging): hedging will have multiple attempts simultaneously.
		committed  bool                       // active attempt committed for retry?
		buffer     []func(a *csAttempt) error // operations to replay on retry
		bufferSize int                        // current size of buffer
	}

client 的构造就更直接了，在 invoke 发起下游调用时， 直接在 sendMsg 之前就会提前构造 clientStream， 如下：

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


stream 这个结构承载了数据流转之外，同时 grpc 流式传输的实现也是基于 stream 去实现的。
