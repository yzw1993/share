## grpc-go

grpc是HTTP/2 RPC的框架。基于HTTP/2标准

### client 

* client的基本行为:

 * 1.建立连接

 * 2.生成client 

```	
// 调用newHTTP2Client()
conn,err := grpc.Dail(serverAddr, opts)  
		
// client 为proto buff 的封装。包含conn结构
client := pb.NewXXXClient(conn)

// 调用某个具体rpc
...			

```

newHTTP2Client 发送和接受数据.将接受的数据填入每个rpc的stream中

```
// 设置默认的dialer
if opts.Dialer == nil {
	opts.Dialer = func(addr string, timeout time.Duration) (net.Conn,error) {
		return net.DialTimeout("tcp",addr,timeout)
	}
}
conn,connErr := opts.Dialer(addr,timeout)

// send http2 header for server 
n,err := conn.Write(clientPreface)	

// 创建http2 reader,writer  
framer := newFramer(conn) 

// 创建http2Client 
t := &http2Client{
	conn: 			conn,
	framer: 		framer,  
	// client streams起始id为1
	nextId:			1,
	activeStreams: 	make(map[uint32]*Stream), 
	// only one 
	writableChan:	make(chan int,1),
}

// http2Client控制信息处理
go t.controller()

// 同时只能有一个writer 进入transport	
t.writableChan <- 0

// 接受in message.将数据copy到stream中
// for {
//     frame,err := t.framer.readFrame() 
// 	   switch frame := frame.(type) {
//         case *.... 
// 		   case *http2.DataFrame:
// 				// s, ok := t.getStream(f)
// 				// copy(data, f.Data())
// 				// s.write(recvMsg{data: data})
// 		        t.handleData(frame)
//     }
// }
go t.reader()
```

### 总结client功能

protobuff中定义service xxx。service xxx被proto buff封装为XXXClient{conn *grpc.ClientConn}。grpc.ClientCon被封装为ClientConn{transport transport.ClientTransport}。ClientTransport为Interface{},对应的实际结构为http2Client。这才是grpc中实际存储conn等所有信息的结构。但http2Client中数据的读写却是http2Client.framer。每个rpc call时都会生成stream并且在http2Client.activeStreams中注册。server返回frame信息中有stream_id以此确定具体rpc。frame将数据copy到rpc对应的stream中。

```
type http2Client struct {
	target string   // server name/addr
	conn   net.Conn // underlying communication channel
	nextID uint32   // the next stream ID to be used

	// writableChan synchronizes write access to the transport.
	// A writer acquires the write lock by sending a value on writableChan
	// and releases it by receiving from writableChan.
	writableChan chan int
	// shutdownChan is closed when Close is called.
	// Blocking operations should select on shutdownChan to avoid
	// blocking forever after Close.
	// TODO(zhaoq): Maybe have a channel context?
	shutdownChan chan struct{}
	// errorChan is closed to notify the I/O error to the caller.
	errorChan chan struct{}

	framer *framer
	hBuf   *bytes.Buffer  // the buffer for HPACK encoding
	hEnc   *hpack.Encoder // HPACK encoder

	// controlBuf delivers all the control related tasks (e.g., window
	// updates, reset streams, and various settings) to the controller.
	controlBuf *recvBuffer
	fc         *inFlow
	// sendQuotaPool provides flow control to outbound message.
	sendQuotaPool *quotaPool
	// streamsQuota limits the max number of concurrent streams.
	streamsQuota *quotaPool

	// The scheme used: https if TLS is on, http otherwise.
	scheme string

	authCreds []credentials.Credentials

	mu            sync.Mutex     // guard the following variables
	state         transportState // the state of underlying connection
	activeStreams map[uint32]*Stream
	// The max number of concurrent streams
	maxStreams int
	// the per-stream outbound flow control window size set by the peer.
	streamSendQuota uint32
}
```
### server

* 1.创建一个proto定义的,与service相关的server

* 2.server注册service的所有方法 

* 3.开始监听 

```
// type Server struct {
//	   opts  options
//     mu    sync.Mutex
// 	   lis   map[net.Listener]bool
//     conns map[transport.ServerTransport]bool
//     m     map[string]*service // service name -> service info
// }
grpcServer := grpc.NewServer(opts...) 

// 构建service信息存入grpcServer   
// for i := range sd.Methods { // rpc 普通
//     d := &sd.Methods[i]
//     srv.md[d.MethodName] = d
// }
// for i := range sd.Streams { // stream rpc
//     d := &sd.Streams[i]
//     srv.sd[d.StreamName] = d
// }
// s.m[sd.ServiceName] = srv  // service name -> service info
pb.RegisterXXXServer(grpcServer, newServer()) 

// 开始监听
// for {
//     c, err := lis.Accept() 
//     st, err := transport.NewServerTransport("http2", c, s.opts.maxConcurrentStreams) 
//     s.conns[st] = true // 一个conn，一个transport 
//     go func() {
//         st.HandleStreams(func(stream *transport.Stream) { // create stream && handle && send resp
//         s.handleStream(st, stream)
//         })
// 		   delete(s.conns, st)
//     }()
// }
grpcServer.Serve(lis)
``` 

#### func (t *http2Server) HandleStreams(handle func(*Stream)){}  

* 1.为rpc在http2Server创建一个stream并且save  
* 2.从client读取REQ message copy至1中创建的stream
* 3.执行下面的handle函数

#### func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream){}

* 1.执行rpc handle
* 2.返回resp

```
// Unary RPC or Streaming RPC?
if md, ok := srv.md[method]; ok {
	s.processUnaryRPC(t, stream, srv, md)
    return
}
if sd, ok := srv.sd[method]; ok {
	s.processStreamingRPC(t, stream, srv, sd)
    return
}
```
#### func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, md *MethodDesc) 

* 1.从stream获取Req 

* 2.执行methodDesc.handle获得resp 

* 3.通过ServerTransport发送resp 

``` 
p := &parser{s: stream}
for {
	pf, req, err := p.recvMsg() 
	if err == io.EOF {
	// The entire stream is done (for unary RPC only). 
		return
	}
	ppErr := md.Handler(srv.server, stream.Context(), s.opts.codec, req)
	s.sendResponse(t, stream, reply, compressionNone, opts)
}
```

#### func (s *Server) processStreamingRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, sd *StreamDesc) 

* 1.将ServerTransport,Stream封装为ServerStream.

* 2.执行handle  

```
s := &serverStream{
    t:     t,
    s:     stream,
    p:     &parser{s: stream},
    codec: s.opts.codec,
}
// resp 通过 t.Write() 发送
sd.Handler(srv.server, ss)

```

### Server总结
 
创建Server,为每个conn创建一个transport并保持在Server中。执行rpc function。将resp 通过transport发送给client 

### 还有transport close()的问题，这个待老夫在抓磨抓膜
