---
title: "net.http Context超时控制"
date: 2021-07-14T21:47:32+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - Golang基础
---


首先看一下使用 net/http 包实现的server和client的简单例子

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"time"
)

func main() {
	ctx, _ := context.WithTimeout(context.Background(), time.Second)
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, "http://localhost", nil)
	if err != nil {
		fmt.Printf("err: %v\n", err)
		return
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		fmt.Printf("err: %v\n", err)
		return
	}

	bytes, err := io.ReadAll(resp.Body)
	if err != nil {
		fmt.Printf("err: %v\n", err)
		return
	}
	fmt.Printf("%s\n", bytes)
}
```

<!--more-->

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", indexHandler)
	http.ListenAndServe("", mux)
}

func indexHandler(w http.ResponseWriter, r *http.Request) {
	start := time.Now()
	done := make(chan int, 1)

	go func() {
		time.Sleep(2 * time.Second)
		done <- 1
	}()

	select {
	case <-r.Context().Done():
		fmt.Println("超时了", time.Since(start).Milliseconds())
	case <-done:
		fmt.Println("正常结束了")
	}
	w.Write([]byte("index"))
}
```

上面的逻辑很简单，client请求server超时时间1s，server处理请求时间2s，很明显context会超时

client运行结果

```go
err: Get "http://localhost": context deadline exceeded
```

server运行结果

```go
超时了 999
```

那么问题来了，从client到server context的超时是怎么传递的呢？

# client端

首先我们构造一个带有超时的context，然后调用http.DefaultClient.Do(req)发送请求

```go
ctx, _ := context.WithTimeout(context.Background(), time.Second)
req, err := http.NewRequestWithContext(ctx, http.MethodGet, "http://localhost", nil)
if err != nil {
	fmt.Printf("err: %v\n", err)
	return
}

resp, err := http.DefaultClient.Do(req)
```

DefaultClient内部调用send方法发送请求，注意这里的deadline是send的超时，并不是我们讨论的context超时

```go
func (c *Client) Do(req *Request) (*Response, error) {
	return c.do(req)
}

func (c *Client) do(req *Request) (retres *Response, reterr error) {
	...
	for {
		if resp, didTimeout, err = c.send(req, deadline); err != nil {
			...
		}
	}
	...
}

func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
	...
	resp, didTimeout, err = send(req, c.transport(), deadline)
	...
}
```

send方法内部调用RoundTripper#RoundTrip()方法发送请求

```go
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
	...
	resp, err = rt.RoundTrip(req)
	...
}

func (t *Transport) RoundTrip(req *Request) (*Response, error) {
	return t.roundTrip(req)
}
```

RoundTrip()方法内部通过getConn方法获取一个pconn对象，最后调用pconn.roundTrip()方法

```go
func (t *Transport) roundTrip(req *Request) (*Response, error) {
	for {
		...
		pconn, err := t.getConn(treq, cm)
		...
		if pconn.alt != nil {
			// HTTP/2 path.
			t.setReqCanceler(cancelKey, nil) // not cancelable with CancelRequest
			resp, err = pconn.alt.RoundTrip(req)
		} else {
			resp, err = pconn.roundTrip(treq)
		}
		...
	}
}
```

pconn.roundTrip()方法内部获取req的context#Done的channal，如果context超时，会执行Transport#cancelRequest()方法

```go
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {

	if !pc.t.replaceReqCanceler(req.cancelKey, pc.cancelRequest) {
		pc.t.putOrCloseIdleConn(pc)
		return nil, errRequestCanceled
	}
	...
	ctxDoneChan := req.Context().Done()

	for {
		select {
		case err := <-writeErrCh:
			...
		case <-pcClosed:
			...
		case <-respHeaderTimer:
			...
		case re := <-resc:
			...
		case <-cancelChan:
			...
		case <-ctxDoneChan:
			canceled = pc.t.cancelRequest(req.cancelKey, req.Context().Err())
			cancelChan = nil
			ctxDoneChan = nil
		}
	}
}
```

Transport#cancelRequest()方法会调用cannel()方法

```go
func (t *Transport) cancelRequest(key cancelKey, err error) bool {
	t.reqMu.Lock()
	cancel := t.reqCanceler[key]
	delete(t.reqCanceler, key)
	t.reqMu.Unlock()
	if cancel != nil {
		cancel(err)
	}

	return cancel != nil
}
```

此时的cannel()方法是persistConn#cancelRequest

```go
func (pc *persistConn) cancelRequest(err error) {
	pc.mu.Lock()
	defer pc.mu.Unlock()
	pc.canceledErr = err
	pc.closeLocked(errRequestCanceled)
}

func (pc *persistConn) closeLocked(err error) {
	...
	if pc.closed == nil {
		pc.closed = err
	
		if pc.alt == nil {
			if err != errCallerOwnsConn {
				pc.conn.Close()
			}
			close(pc.closech)
		}
	}
	pc.mutateHeaderFunc = nil
}
```

可以看到，最后会调用pc.conn.Close()方法将conn关闭

总结：client端如果context超时了，会主动将conn进行close

# Server端

服务端启动，调用http.ListenAndServe()方法

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", indexHandler)
	http.ListenAndServe("", mux)
}
```

ListenAndServe()方法会监听tcp端口，并为每个conn创建1个goroutine进行处理

```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}

func (srv *Server) ListenAndServe() error {
	...
	ln, err := net.Listen("tcp", addr)
	...
	return srv.Serve(ln)
}

func (srv *Server) Serve(l net.Listener) error {
	...
	baseCtx := context.Background()
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept()
		...
		go c.serve(connCtx)
	}
}
```

serve()方法是个死循环，在循环内部从conn中读取出request对象，在将request对象交给ServerHTTP方法(也就是我们的业务handler)之前，调用了startBackgroundRead()方法

```go
func (c *conn) serve(ctx context.Context) {
	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
	...
	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx // 这里是重点1
	defer cancelCtx()

	for {
		w, err := c.readRequest(ctx)
		...
		if err != nil {
			switch {
			...
			case isCommonNetReadError(err):
				return // don't reply
			}
			...
		}

		if requestBodyRemains(req.Body) {
			registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
		} else {
			w.conn.r.startBackgroundRead() // 这里是重点2
		}

		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.cancelCtx()
	}
}
```

startBackgroundRead()方法中再创建了一个goroutine

```go
func (cr *connReader) startBackgroundRead() {
	...
	go cr.backgroundRead()	
}
```

这个goroutine从conn中继续阻塞读取数据

```go
func (cr *connReader) backgroundRead() {
	n, err := cr.conn.rwc.Read(cr.byteBuf[:])
	if ne, ok := err.(net.Error); ok && cr.aborted && ne.Timeout() {
		// Ignore this error. It's the expected error from
		// another goroutine calling abortPendingRead.
	} else if err != nil {
		cr.handleReadError(err)
	}
	...
}
```

再前面分析client端的时候知道，context超时的时候会将conn进行close，如果再从conn里面读取数据就会返回err，然后调用handleReadError()方法

```go
func (cr *connReader) handleReadError(_ error) {
	cr.conn.cancelCtx()
	cr.closeNotify()
}
```

handleReadError()方法内部会调用conn.cannelCtx()，在代码注释的重点1部分对conn.cannelCtx进行了赋值，这个cannelCtx调用就会影响到我们业务中的context，进而控制我们业务的超时

到此并没有结束，当context超时后就会从serverHandler{c.server}.ServeHTTP(w, w.req)调用返回

```go
func (c *conn) serve(ctx context.Context) {
	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
	...
	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx // 这里是重点1
	defer cancelCtx()

	for {
		w, err := c.readRequest(ctx)
		...
		if err != nil {
			switch {
			...
			case isCommonNetReadError(err):
				return // don't reply
			}
			...
		}

		if requestBodyRemains(req.Body) {
			registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
		} else {
			w.conn.r.startBackgroundRead() // 这里是重点2
		}

		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.cancelCtx()
	}
}
```

因为是在for循环中，会继续调用c.readRequest(ctx)方法从conn中读取数据，此时conn已经close就会返回err，最后会到isCommonNetReadError(err)退出整个for循环

# 总结

client端通过context超时将conn关闭，server端通过从conn读取数据出现 err 将context进行cancel