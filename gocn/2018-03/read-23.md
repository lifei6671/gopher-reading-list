## fasthttp的goroutine pool实现探究

### 1. 引言

fasthttp是一个非常优秀的web server框架，号称比官方的net/http快10倍以上。fasthttp用了很多黑魔法。俗话说，源码面前，了无秘密，我们今天通过源码来看一看她的goroutine pool的实现。

### 2. 热身

fasthttp写server和原生的net/http写法上基本没有区别，这里就不举例子。直接找到入口函数，在根目录下的server.go文件中，我们从函数`ListenAndServe()`跟踪进去。从端口监听到处理请求的函数调用链如下。

```go
func ListenAndServe(addr string, handler RequestHandler) error {
    s := &Server{
        Handler: handler,
    }
    return s.ListenAndServe(addr)
}

// ListenAndServe serves HTTP requests from the given TCP addr.
func (s *Server) ListenAndServe(addr string) error {
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return s.Serve(ln)
}

// Serve blocks until the given listener returns permanent error.
func (s *Server) Serve(ln net.Listener) error {
    ...
    wp := &workerPool{
        WorkerFunc:      s.serveConn,
        MaxWorkersCount: maxWorkersCount,
        LogAllErrors:    s.LogAllErrors,
        Logger:          s.logger(),
    }
    wp.Start()  //启动worker pool
    
    for {
        if c, err = acceptConn(s, ln, &lastPerIPErrorTime); err != nil {
            wp.Stop()
            if err == io.EOF {
                return nil
            }
            return err
        }
        if !wp.Serve(c) {
            s.writeFastError(c, StatusServiceUnavailable,
                "The connection cannot be served because Server.Concurrency limit exceeded")
            c.Close()
            if time.Since(lastOverflowErrorTime) > time.Minute {
                s.logger().Printf("The incoming connection cannot be served, because %d concurrent connections are served. "+
                    "Try increasing Server.Concurrency", maxWorkersCount)
                lastOverflowErrorTime = time.Now()
            }
            time.Sleep(100 * time.Millisecond)
        }
        c = nil
    }
}
```

上面代码中workerPool就是一个线程池。相关代码在server.go文件的同级目录下的workerpool.go文件中。我们从上面代码涉及到的往下看。首先是`workerPool struct`。

```go
type workerPool struct {
    WorkerFunc func(c net.Conn) error
    MaxWorkersCount int
    LogAllErrors bool
    MaxIdleWorkerDuration time.Duration
    Logger Logger
    lock         sync.Mutex
    workersCount int
    mustStop     bool
    ready []*workerChan

    stopCh chan struct{}
    workerChanPool sync.Pool
}

type workerChan struct {
    lastUseTime time.Time
    ch          chan net.Conn
}
```

`workerPool sturct`中的WorkerFunc是conn的处理函数，类似net/http包中的ServeHTTP。因为所有conn的处理都是一样的，所以WorkerFunc不需要和传入的每个conn绑定，整个worker pool共用一个。`workerChanPool`是`sync.Pool`对象池。`MaxIdleWorkerDuration`是worker空闲的最长时间，超过就将worker关闭。`workersCount`是worker的数量。ready是可用的worker列表，也就是说所有goroutine worker是存放在一个数组里面的。这个数组模拟一个类似栈的FILO队列，也就是说我们每次使用的worker都从队列的尾部开始取。`wp.Start()`启动worker pool。`wp.Stop()`是出错处理。`wp.Serve(c)`是对conn进行处理的函数。我们先看一下`wp.Start()`。

```go
func (wp *workerPool) Start() {
    if wp.stopCh != nil {
        panic("BUG: workerPool already started")
    }
    wp.stopCh = make(chan struct{})
    stopCh := wp.stopCh
    go func() {
        var scratch []*workerChan
        for {
            wp.clean(&scratch)
            select {
            case <-stopCh:
                return
            default:
                time.Sleep(wp.getMaxIdleWorkerDuration())
            }
        }
    }()
}

func (wp *workerPool) Stop() {
    ...
    close(wp.stopCh)
    wp.stopCh = nil

    wp.lock.Lock()
    ready := wp.ready
    for i, ch := range ready {
        ch.ch <- nil
        ready[i] = nil
    }
    wp.ready = ready[:0]
    wp.mustStop = true
    wp.lock.Unlock()
}
```

简单来说，`wp.Start()`启动了一个goroutine，负责定期清理worker pool中过期worker(过期=未使用时间超过MaxIdleWorkerDuration)。清理操作都在`wp.clean()`函数中完成，这里就不继续往下看了。stopCh是一个标示worker pool停止的chan。上面的for-select-stop是很常用的方式。wp.Stop()负责停止worker pool的处理工作，包括关闭stopCh，清理闲置的worker列表（这时候还有一部分worker在处理conn，待其处理完成通过判断wp.mustStop来停止）。这里需要注意的一点是做资源清理的时候，对于channel需要置nil。下面看看最重要的函数`wp.Serve()`。

### 3. 核心

下面是`wp.Serve()`函数的调用链。`wp.Serve()`负责处理来自client的每一条连接。其中比较关键的函数是`wp.getCh()`，她从worker pool的可用空闲worker列表尾部取出一个可用的worker。这里有几个逻辑需要注意的是：1.如果没有可用的worker（比如处理第一个conn是，worker pool还是空的）则新建；2.如果worker达到上限，则直接不处理（这个地方感觉略粗糙啊！）。`go func()`那几行代码就是新建worker，我们放到下面说。

```go
func (wp *workerPool) Serve(c net.Conn) bool {
    ch := wp.getCh()
    if ch == nil {
        return false
    }
    ch.ch <- c
    return true
}

func (wp *workerPool) getCh() *workerChan {
    var ch *workerChan
    createWorker := false

    wp.lock.Lock()
    ready := wp.ready
    n := len(ready) - 1
    if n < 0 {
        if wp.workersCount < wp.MaxWorkersCount {
            createWorker = true
            wp.workersCount++
        }
    } else {
        ch = ready[n]
        ready[n] = nil
        wp.ready = ready[:n]
    }
    wp.lock.Unlock()

    if ch == nil {
        if !createWorker {
            return nil
        }
        vch := wp.workerChanPool.Get()
        if vch == nil {
            vch = &workerChan{
                ch: make(chan net.Conn, workerChanCap),
            }
        }
        ch = vch.(*workerChan)
        go func() {
            wp.workerFunc(ch)
            wp.workerChanPool.Put(vch)
        }()
    }
    return ch
}
```

`workerFunc()`函数定义如下(去掉了很多不影响主线的逻辑)，结合上一篇[《如何裸写一个goroutine pool》](http://legendtkl.com/2016/09/06/go-pool/)，还是熟悉的配方，熟悉的味道。这里要看的`wp.release()`是干啥的。因为前面的`wp.Serve()`函数只处理一个conn，所以for循环执行一次我们就可以把worker放到空闲队列中去等待下一次conn过来，从代码中可以看出来放回果然是放到空闲队列的末尾（可算和上面呼应上了）。还有上面提到的mustStop，如果worker pool停止了，mustStop就为true，那么workerFunc就要跳出循环，也就是goroutine结束了。


```go
func (wp *workerPool) workerFunc(ch *workerChan) {
    var c net.Conn

    var err error
    for c = range ch.ch {
        if c == nil {
            break
        }

        ...
        c = nil

        if !wp.release(ch) {
            break
        }
    }

    wp.lock.Lock()
    wp.workersCount--
    wp.lock.Unlock()
}

func (wp *workerPool) release(ch *workerChan) bool {
    ch.lastUseTime = time.Now()
    wp.lock.Lock()
    if wp.mustStop {
        wp.lock.Unlock()
        return false
    }
    wp.ready = append(wp.ready, ch)
    wp.lock.Unlock()
    return true
}
```


### 4. 总结

除了fasthttp，我还看了github上其他开源且star数在100以上的goroutine pool的实现，基本核心原理都在我上一篇文章中说的那些。fasthttp的实现多了一层goroutine回收机制，不得不说确实挺巧妙。fasthttp性能这么好一定是有其原因的，源码之后再慢慢读。

#

    作者：legendtkl
    链接：http://legendtkl.com/2016/09/09/fasthttp-inside/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。





























