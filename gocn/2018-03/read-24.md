## 如何裸写一个goroutine pool

### 1. 引言

在上文中，我说到golang的原生http server处理client的connection的时候，每个connection起一个goroutine，这是一个相当粗暴的方法。为了感受更深一点，我们来看一下go的源码。先定义一个最简单的http server如下。

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello there!\n")
}

func main(){
    http.HandleFunc("/", myHandler)     //  设置访问路由
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

从入口http.ListenAndServe函数跟进去。

```go
// file: net/http/server.go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}

func (srv *Server) ListenAndServe() error {
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})        
}

func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()
    ...
    for {
        rw, e := l.Accept()
        if e != nil {
            // error handle
            return e
        }
        tempDelay = 0
        c, err := srv.newConn(rw)
        if err != nil {
            continue
        }
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve()
    }
}
```

首先`net.Listen`负责监听网络端口，`rw, e := l.Accept()`则从网络端口中取出TCP连接，然后`go c.server()`则对每一个TCP连接起一个goroutine来处理。我还说到fasthttp这个网络框架性能要比原生的net/http性能要好，其中一个原因就是因为使用了goroutine pool。那么问题来了，如果要我们自己去实现一个goroutine pool，该怎么去实现呢？我们先来实现一个最简单的。

### 2. 弱鸡版

golang中的goroutine通过go来启动，goroutine资源和临时对象池不一样，不能放回去再取出来。所以goroutine应该是一直运行着的。需要的时候就运行，不需要的时候就阻塞，这样对其他的goroutine的调度影响也不是很大。而goroutine的任务可以通过channel来传递就ok了。很简单的弱鸡版本就出来了，如下。

```go
func Gopool() {
    start := time.Now()
    wg := new(sync.WaitGroup)
    data := make(chan int, 100)

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            for _ = range data {
                fmt.Println("goroutine:", n, i)
            }
        }(i)
    }

    for i := 0; i < 10000; i++ {
        data <- i
    }
    close(data)
    wg.Wait()
    end := time.Now()
    fmt.Println(end.Sub(start))
}
```

上面的代码中还做了程序运行时间统计。作为对比，下面是一个没有使用pool的版本。

```go
func Nopool() {
    start := time.Now()
    wg := new(sync.WaitGroup)

    for i := 0; i < 10000; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            //fmt.Println("goroutine", n)
        }(i)
    }
    wg.Wait()

    end := time.Now()
    fmt.Println(end.Sub(start))
}
```

最后运行时间对比，使用了goroutine pool的代码运行时间约为没有使用pool的代码的2/3。当然这么测试还是略显粗糙了。我们下面使用reflect那篇文章里面介绍的go benchmark testing的方式测试，测试代码如下(去掉了很多无关代码)。

```go
package pool

import (
    "sync"
    "testing"
)

func Gopool() {
    wg := new(sync.WaitGroup)
    data := make(chan int, 100)

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            for _ = range data {
            }
        }(i)
    }

    for i := 0; i < 10000; i++ {
        data <- i
    }
    close(data)
    wg.Wait()
}

func Nopool() {
    wg := new(sync.WaitGroup)

    for i := 0; i < 10000; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
        }(i)
    }
    wg.Wait()
}

func BenchmarkGopool(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Gopool()
    }
}

func BenchmarkNopool(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Nopool()
    }
}
```

最终的测试结果如下，使用了goroutine pool的代码执行时间确实更短。

```shell
$ go test -bench='.' gopool_test.go
BenchmarkGopool-8            500       2596750 ns/op
BenchmarkNopool-8            500       3604035 ns/op
PASS
```

### 3. 升级版

对于一个好的线程池，我们往往有更多的需求，一个最迫切的需求是能自定义goroutine运行的函数。函数无非就是函数地址和函数参数。如果要传入的函数形式不一样（形参或者返回值不一样）怎么办？一个比较简单的方法是引入反射。


```go
type worker struct {
    Func interface{}
    Args []reflect.Value
}

func main() {
    var wg sync.WaitGroup

    channels := make(chan worker, 10)
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for ch := range channels {
                reflect.ValueOf(ch.Func).Call(ch.Args)
            }
        }()
    }

    for i := 0; i < 100; i++ {
        wk := worker{
            Func: func(x, y int) {
                fmt.Println(x + y)
            },
            Args: []reflect.Value{reflect.ValueOf(i), reflect.ValueOf(i)},
        }
        channels <- wk
    }
    close(channels)
    wg.Wait()
}
```

但是引入反射又会引入性能问题。本来goroutine pool就是为了解决性能问题，然而现在又引入了新的性能问题。那么怎么办呢？闭包。

```go
type worker struct {
    Func func()
}

func main() {
    var wg sync.WaitGroup

    channels := make(chan worker, 10)

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for ch := range channels {
                //reflect.ValueOf(ch.Func).Call(ch.Args)
                ch.Func()
            }
        }()
    }

    for i := 0; i < 100; i++ {
        j := i
        wk := worker{
            Func: func() {
                fmt.Println(j + j)
            },
        }
        channels <- wk
    }
    close(channels)
    wg.Wait()
}
```

这里值得注意的一点是golang的闭包用不好容易把自己代入坑，而理解闭包一个很关键的点就是对对象的引用而不是复制。这里只是goroutine pool 实现的一个精简版，真正实现的时候还有很多细节需要考虑，比如设置一个stop channel用来停止pool，但是goroutine pool的核心就在于这个地方。

### 4. goroutine池和CPU核的关系

那么goroutine池里面goroutine数目和核数有没有关系呢？这个其实要分情况讨论。

#### 1.goroutine池跑不满

这也就意味着channel data里面一有数据就会被goroutine拿走，这样的话当然只能你CPU能调度的过来就行，也就是池子里的goroutine数目和CPU核数是最优的。经测试，确实是这样。

#### 2.channel data有数据阻塞

这意思是说goroutine是不够用的，如果goroutine的运行任务不是CPU密集型的（大部分情况都不是），而只是IO阻塞，这个时候一般goroutine数目在一定范围内是越多越好，当然范围在什么地方就要具体情况具体分析了。


#

    作者：legendtkl
    链接：http://legendtkl.com/2016/09/06/go-pool/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。































