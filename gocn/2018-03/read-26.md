## go编写web server的几种方式

先说一下web server和http server的区别。http server，顾名思义，支持http协议的服务器；web server除了支持http协议可能还支持其他网络协议。本文只讨论使用golang的官方package编写web server的几种常用方式。

### 最简单的http server

这也是最简单的一种方式。


```go
package main

import (
    "net/http"
    "log"  
)

func myHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello there!\n")
}

func main(){
    http.HandleFunc("/", myHandler)		//	设置访问路由
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```


启动程序，另开一个终端输入命令`curl localhost:8080`，或者直接浏览器打开localhost:8080，就可以看到Hello there!

ListenAndServe函数负责监听并处理连接。内部处理方式是对于每个connection起一个goroutine来处理。其实这并不是一种好的处理方式。学习过操作系统的同学都知道，进程或者线程切换的代价是巨大的。虽然goroutine是用户级的轻量级线程，切换并不会导致用户态和内核态的切换，但是当goroutine数量巨大的时候切换的代价还是不容小觑的。更好的一种方式是使用goroutine pool，这里暂且按住不表。

### 使用Handler接口

上面这种方式感觉可发挥余地太小了，比如我想设置server的Timeout时间都不能设置了。这时候我们就可以使用自定义server了。

```go
type Server struct {
    Addr		string		//TCP address to listen on
    Handler		Handler		//handler to invoke
    ReadTimeout	time.Duration	//maximum duration before timing out read of the request
    WriteTimeout time.Duration	//maximum duration before timing out write of the response
    TLSConfig	*tls.Config
    ...
}
//Handler是一个interface，定义如下
type Handler interface {
    ServeHTTP(ResponseWrite, *Request)
}
```

所以只要我们实现了Handler接口的方法ServeHTTP就可以自定义我们的server了。示例代码如下。

```go
type myHandler struct{
  
}

func (this myHandler) ServeHTTP(w ResponseWrite, r *Request) {
    ...
}

func main() {
    server := http.Server{
        Addr:	":8080",
        Handler:	&myHandler{},
        ReadTimeout:	3*time.Second,
        ...
    }
    log.Fatal(server.ListenAndServe)
}
```

### 直接处理conn

有时候我们需要更底层一点直接处理connection，这时候可以使用net包。server端代码简单实现如下。

```go
func main() {
    listener, err := net.Listen("tcp", ":8080")
    if err != nil {
        log.Fatal(err)	
    }
  
    for {
        conn, err := listener.Accept()
        if err != nil {
            //handle error
        }
        go handleConn(conn)
    }
}
```

为了示例方便，我这里对于每一个connection都起了一个goroutine去处理。实际使用中，goroutine pool往往是一个更好的选择。对于client的返回信息我们再歇回到conn中就可以。

### proxy

上面说了几种web server的实现方式，下面简单实现一个http proxy，用golang写proxy很简单只需要把conn转发就可以了。

```go
//代理服务器
func main() {
    listener, err := net.Listen("tcp", ":8080")
    if err != nil {
        log.Fatal(err)	
    }
  
    for {
        conn, err := listener.Accept()
        if err != nil {
            //handle error
        }
        go handleConn(conn)
    }
}

func handleConn(from net.Conn) {
    to, err := net.Dial("tcp", ":8001")	//建立和目标服务器的连接
    if err != nil {
        //handle error
    }
  
    done := make(chan struct{})
  
    go func() {
        defer from.Close()
        defer to.Close()
        io.Copy(from, to)
        done<-strcut{}{}
    }()
  
    go func() {
        defer from.Close()
        defer to.Close()
        io.Copy(to, from)
        done<-struct{}{}
    }
  
    <-done
    <-done
}
```

proxy稍微强化一点就可以实现#不可描述#的目的了。

#

    作者：legendtkl
    链接：http://legendtkl.com/2016/08/21/go-web-server/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。




























