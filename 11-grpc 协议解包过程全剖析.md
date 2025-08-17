> 本文原发表于：https://diu.life/lessons/grpc-read/grpc-protocol-unpacking-analysis/
> 最新版本请访问原文链接

###  http2 协议帧格式

我们知道网络传输都是以二进制的形式，所以所有的协议底层的传输也是二进制。那么问题来了，client 往 server 发一个数据包，server 如何知道数据包是完成还是还在发送呢？又或者，假如一个数据包过大，client 需要拆成几个包发送，或者数据包过小，client 需要合成一个包发送，server 如何识别呢？为了解决这些问题，client 和 server 都会约定好一些双方都能理解的“规则“，这就是协议。

我们知道 grpc 传输层是基于 http2 协议规范，所以我们先要了解 http2 协议帧的格式。http 2 协议帧格式如下：

```http2
Frame Format
All frames begin with a fixed 9-octet header followed by a variable-length payload.

 +-----------------------------------------------+
 |                 Length (24)                   |
 +---------------+---------------+---------------+
 |   Type (8)    |   Flags (8)   |
 +-+-------------+---------------+-------------------------------+
 |R|                 Stream Identifier (31)                      |
 +=+=============================================================+
 |                   Frame Payload (0...)                      ...
 +---------------------------------------------------------------+
```
对于一个网络包而言，首先要知道这个包的格式，然后才能按照约定的格式解析出这个包。那么 grpc 的包是什么样的格式呢？ 看了源码后，先直接揭晓出来，它其实是这样的

![](https://images.xiaozhuanlan.com/photo/2019/ad81643a987d5ae267f3ea2dc4cd3434.png)

http 帧格式为：length (3 byte) + type(1 byte) + flag (1 byte)  + R (1 bit) + stream identifier  (31 bit) + paypoad，payload 是消息具体内容

前 9 个字节是 http 包头，length 表示消息长度，type 表示 http 帧的类型，http 一共规定了 10 种帧类型：

- HEADERS帧 头信息，对应于HTTP HEADER
- DATA帧 对应于HTTP Response Body
- PRIORITY帧 用于调整流的优先级
- RST_STREAM帧 流终止帧，用于中断资源的传输
- SETTINGS帧 用户客户服务器交流连接配置信息
- PUSH_PROMISE帧 服务器向客户端主动推送资源
- GOAWAY帧 通知对方断开连接
- PING帧 心跳帧，检测往返时间和连接可用性
- WINDOW_UPDATE帧 调整帧大小
- CONTINUATION帧 HEADERS太大时的续帧

flag 表示标志位，http 一共三种标志位：

- END_STREAM 流结束标志，表示当前帧是流的最后一个帧
- END_HEADERS 头结束表示，表示当前帧是头信息的最后一个帧
- PADDED 填充标志，在数据Payload里填充无用信息，用于干扰信道监听

R 是 1bit 的保留位，stream identifier 是流 id，http 会为每一个数据流分配一个 id

具体可以参考：[http frame](https://http2.github.io/http2-spec/#FramingLayer)


### 解析 http 帧头

回到 examples 目录的 helloworld 目录， 在 server main 函数中跟踪 s.Serve 方法： s.Serve() ——> s.handleRawConn(rawConn) ——> s.serveStreams(st) 

来看一下 serveStreams 这个方法 

```go
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
```
这里调用了 transport 的 HandleStreams 方法， 这个方法就是 http 帧的处理的具体实现。它的底层直接调用的 http2 包的 ReadFrame 方法去读取一个 http 帧数据。

```go
type framer struct {
	writer *bufWriter
	fr     *http2.Framer
}

func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
	defer close(t.readerDone)
	for {
		frame, err := t.framer.fr.ReadFrame()
		atomic.StoreUint32(&t.activity, 1)
		
		...
		
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
```

通过 http2 包的 ReadFrame 直接读取出一个帧的数据。

```go
func (fr *Framer) ReadFrame() (Frame, error) {
	fr.errDetail = nil
	if fr.lastFrame != nil {
		fr.lastFrame.invalidate()
	}
	fh, err := readFrameHeader(fr.headerBuf[:], fr.r)
	if err != nil {
		return nil, err
	}
	if fh.Length > fr.maxReadSize {
		return nil, ErrFrameTooLarge
	}
	payload := fr.getReadBuf(fh.Length)
	if _, err := io.ReadFull(fr.r, payload); err != nil {
		return nil, err
	}
	f, err := typeFrameParser(fh.Type)(fr.frameCache, fh, payload)
	if err != nil {
		if ce, ok := err.(connError); ok {
			return nil, fr.connError(ce.Code, ce.Reason)
		}
		return nil, err
	}
	if err := fr.checkFrameOrder(f); err != nil {
		return nil, err
	}
	if fr.logReads {
		fr.debugReadLoggerf("http2: Framer %p: read %v", fr, summarizeFrame(f))
	}
	if fh.Type == FrameHeaders && fr.ReadMetaHeaders != nil {
		return fr.readMetaFrame(f.(*HeadersFrame))
	}
	return f, nil
}
```
fh, err := readFrameHeader(fr.headerBuf[:], fr.r)  这一行代码读取了 http 的包头数据，我们来看一下 headerBuf 的长度，发现果然是 9 个字节。

```go
const frameHeaderLen = 9

func readFrameHeader(buf []byte, r io.Reader) (FrameHeader, error) {
	_, err := io.ReadFull(r, buf[:frameHeaderLen])
	if err != nil {
		return FrameHeader{}, err
	}
	return FrameHeader{
		Length:   (uint32(buf[0])<<16 | uint32(buf[1])<<8 | uint32(buf[2])),
		Type:     FrameType(buf[3]),
		Flags:    Flags(buf[4]),
		StreamID: binary.BigEndian.Uint32(buf[5:]) & (1<<31 - 1),
		valid:    true,
	}, nil
}
```

### 解析业务数据 

经过上面的过程，我们终于将 http 包头给读出来了。前面说到了，读出 http 包体之后，还需要解析 grpc 协议头。那这部分是怎么去解析的呢？

回到 http2 读帧的部分，当发现帧的格式是 MetaHeadersFrame，也就是第一个帧时，会调用 operateHeaders 方法

```go
case *http2.MetaHeadersFrame:
			if t.operateHeaders(frame, handle, traceCtx) {
				t.Close()
				break
			}
```

看一下 operateHeaders ，里面会去调用 handle(s) ， 这个handle 其实是 之前 s.Serve() ——> s.handleRawConn(rawConn) ——> s.serveStreams(st)  这个路径下的 HandleStreams 方法传入的，也就是会去调用 handleStream 这个方法

```
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
```

s.handleStream(st, stream, s.traceInfo(st, stream)) ——> s.processUnaryRPC(t, stream, srv, md, trInfo) ———> d, err := recvAndDecompress(&parser{r: stream}, stream, dc, s.opts.maxReceiveMessageSize, payInfo, decomp)  ，进入 recvAndDecompress 这个函数，里面调用了

```
	pf, d, err := p.recvMsg(maxReceiveMessageSize)
```

进入 recvMsg，发现它就是解析 grpc 协议 的函数，先把协议头读出来，用了 5 个字节。从协议头中得知协议体消息的长度，然后用一个相应长度的 byte 数组把协议体读出来

```go
type parser struct {
	r io.Reader

	header [5]byte
}

func (p *parser) recvMsg(maxReceiveMessageSize int) (pf payloadFormat, msg []byte, err error) {
	if _, err := p.r.Read(p.header[:]); err != nil {
		return 0, nil, err
	}

	pf = payloadFormat(p.header[0])
	length := binary.BigEndian.Uint32(p.header[1:])

	... 
	msg = make([]byte, int(length))
	if _, err := p.r.Read(msg); err != nil {
		if err == io.EOF {
			err = io.ErrUnexpectedEOF
		}
		return 0, nil, err
	}
	
	return pf, msg, nil
}
```

继续回到这个图，前面说到了 grpc 协议头是 5个字节。
![](https://images.xiaozhuanlan.com/photo/2019/ad81643a987d5ae267f3ea2dc4cd3434.png)
compressed-flag 表示是否压缩，值为 1 是压缩消息体数据，0 不压缩。
length 表示消息体数据长度。现在终于知道了这个数据结构的由来！

读取出来的数据是二进制的，读出来原数据之后呢，我们就可以针对相应的数据做解包操作了。这里可以参考我的另一篇文章 ：[10-grpc 协议编解码器](https://github.com/lubanproj/grpc_read/blob/master/10-grpc%20%E5%8D%8F%E8%AE%AE%E7%BC%96%E8%A7%A3%E7%A0%81%E5%99%A8.md)
