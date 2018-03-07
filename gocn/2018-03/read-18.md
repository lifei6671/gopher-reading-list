## Go Channel 源码剖析

### 0. 引言

这篇文章介绍一下 Golang channel 的内部实现，包括 channel 的数据结构以及相关操作的代码实现。代码版本 go1.9rc1，部分无关代码直接略去，比如 race detect，对应的代码中的 raceenabled。

### 1. hchan struct

channel 的底层数据结果是 hchan struct。

```go
type hchan struct {
    qcount   uint           // 队列中数据个数
    dataqsiz uint           // channel 大小
    buf      unsafe.Pointer // 存放数据的环形数组
    elemsize uint16         // channel 中数据类型的大小
    closed   uint32         // 表示 channel 是否关闭
    elemtype *_type // 元素数据类型
    sendx    uint   // send 的数组索引
    recvx    uint   // recv 的数组索引
    recvq    waitq  // 由 recv 行为（也就是 <-ch）阻塞在 channel 上的 goroutine 队列
    sendq    waitq  // 由 send 行为 (也就是 ch<-) 阻塞在 channel 上的 goroutine 队列

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
type waitq struct {
    first *sudog
    last  *sudog
}
type sudog struct {
    // The following fields are protected by the hchan.lock of the
    // channel this sudog is blocking on. shrinkstack depends on
    // this for sudogs involved in channel ops.

    g          *g
    selectdone *uint32 // CAS to 1 to win select race (may point to stack)
    next       *sudog
    prev       *sudog
    elem       unsafe.Pointer // data element (may point to stack)

    // The following fields are never accessed concurrently.
    // For channels, waitlink is only accessed by g.
    // For semaphores, all fields (including the ones above)
    // are only accessed when holding a semaRoot lock.

    acquiretime int64
    releasetime int64
    ticket      uint32
    parent      *sudog // semaRoot binary tree
    waitlink    *sudog // g.waiting list or semaRoot
    waittail    *sudog // semaRoot
    c           *hchan // channel
}
```

上面直接对各个字段做了解释。我们可以看到 channel 其实就是一个队列加一个锁，只不过这个锁是一个轻量级锁。其中 recvq 是读操作阻塞在 channel 的 goroutine 列表，sendq 是写操作阻塞在 channel 的 goroutine 列表。列表的实现是 sudog，其实就是一个对 g 的结构的封装。

### 2. make

通过 make 创建 channel 对应的代码如下。

```go
func makechan(t *chantype, size int64) *hchan {
    elem := t.elem

    // compiler checks this but be safe.
    if elem.size >= 1<<16 {
        throw("makechan: invalid channel element type")
    }
    if hchanSize%maxAlign != 0 || elem.align > maxAlign {
        throw("makechan: bad alignment")
    }
    if size < 0 || int64(uintptr(size)) != size || (elem.size > 0 && uintptr(size) > (_MaxMem-hchanSize)/elem.size) {
        panic(plainError("makechan: size out of range"))
    }

    var c *hchan
    if elem.kind&kindNoPointers != 0 || size == 0 {
        // Allocate memory in one call.
        // Hchan does not contain pointers interesting for GC in this case:
        // buf points into the same allocation, elemtype is persistent.
        // SudoG's are referenced from their owning thread so they can't be collected.
        // TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
        c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
        if size > 0 && elem.size != 0 {
            c.buf = add(unsafe.Pointer(c), hchanSize)
        } else {
            // race detector uses this location for synchronization
            // Also prevents us from pointing beyond the allocation (see issue 9401).
            c.buf = unsafe.Pointer(c)
        }
    } else {
        c = new(hchan)
        c.buf = newarray(elem, int(size))
    }
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)

    if debugChan {
        print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
    }
    return c
}
```

最前面的两个 if 是一些异常判断：元素类型大小限制和对齐限制。第三个 if 也很明显，判断 size 大小是否小于 0 或者过大。`int64(uintptr(size)) != size` 这句也是判断 size 是否为负。值得一说的是最后面的判断条件

```go
uintptr(size) > (_MaxMem-hchanSize)/elem.size

```

`_MaxMem` 我在 [Golang 内存管理](http://legendtkl.com/2017/04/02/golang-alloc/) 那篇文章里面说过，这个是 Arena 区域的最大值，用来分配给堆的。也就是说 channel 是在堆上分配的。

再往下就可以看到分配的代码了。如果 channel 内数据类型不含有指针且 size > 0，则将其分配在连续的内存区域。如果 size = 0，实际上 buf 是不分配空间的。

```go
if elem.kind&kindNoPointers != 0 || size == 0 {
    // Allocate memory in one call.
    // Hchan does not contain pointers interesting for GC in this case:
    // buf points into the same allocation, elemtype is persistent.
    // SudoG's are referenced from their owning thread so they can't be collected.
    // TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
    c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
    if size > 0 && elem.size != 0 {
        c.buf = add(unsafe.Pointer(c), hchanSize)
    } else {
        // race detector uses this location for synchronization
        // Also prevents us from pointing beyond the allocation (see issue 9401).
        c.buf = unsafe.Pointer(c)
    }
}
```

除了上面的情况，剩下的，也就是 size > 0，channel 和 channel.buf 是分别进行分配的。剩下的代码是剩下字段的处理。

```go
else {
        c = new(hchan)
        c.buf = newarray(elem, int(size))   // newarray 也是调用 mallocgc 进行内存分配
}
```

总结一下，make chan 的过程是在堆上进行分配，返回是一个 hchan 的指针。

### 3. send

send 也就是 `ch <- x`，对应的函数如下。

```go
// entry point for c <- x from compiled code
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc(unsafe.Pointer(&c)))
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {
        if !block {
            return false
        }
        gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
        throw("unreachable")
    }
    ...
    
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    lock(&c.lock)

    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }

    if !block {
        unlock(&c.lock)
        return false
    }

    // Block on the channel. Some receiver will complete our operation for us.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.selectdone = nil
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg)
    goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

    // someone woke us up.
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if gp.param == nil {
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
        panic(plainError("send on closed channel"))
    }
    gp.param = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    mysg.c = nil
    releaseSudog(mysg)
    return true
}
```

#### 3.1 nil channel

先来看一下 nil channel 的情况，也就是向没有 make 的 channel 发送数据。上篇文章 深入理解 Go Channel 中留了一个问题：向 nil channel 发送数据会报 fatal error: all goroutines are asleep - deadlock! 错误。

```go
if c == nil {
    gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
    throw("unreachable")
}

//runtime/trace.go
traceEvGoStop            = 16 // goroutine stops (like in select{}) [timestamp, stack]

//runtime/proc.go
// Puts the current goroutine into a waiting state and calls unlockf.
// If unlockf returns false, the goroutine is resumed.
// unlockf must not access this G's stack, as it may be moved between
// the call to gopark and the call to unlockf.
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason string, traceEv byte, traceskip int) {}

```

gopark 会将当前 goroutine 休眠，然后通过 unlockf 来唤醒，注意我们上面传入的 unlockf 是 nil，也就是向 nil channel 发送数据的 goroutine 会一直休眠。同理，从 nil channel 读数据也是一样的处理。我们再看一眼上一篇文章的例子。

```go
func main() {
    var x chan int
    go func() {
        x <- 1
    }()
    <-x
}
```

这里一个是 main goroutin 从 nil channel 读数据，进入休眠。go func() 向 nil channel 发送数据，也进入休眠。然后 Go 语言启动的时候还有一个goroutine sysmon 会一直检测系统的运行情况，比如 checkdead()。

```go
func checkdead() {
    ...
    throw("all goroutines are asleep - deadlock!")  // 错误信息就是这里报出来的。
}
```

#### 3.2 closed channel

向 close 的 channel 发送数据，直接 panic。

```go
lock(&c.lock)

if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("send on closed channel"))
}
```

#### 3.3 发送数据处理

发送数据分三种情况：

- 有 goroutine 阻塞在 channel 上，此时 hchan.buf 为空：直接将数据发送给该 goroutine。
- 当前 hchan.buf 还有可用空间：将数据放到 buffer 里面。
- 当前 hchan.buf 已满：阻塞当前 goroutine。

第一种情况如下。从当前 channel 的等待队列中取出等待的 goroutine，然后调用 send。goready 负责唤醒 goroutine。

```go
lock(&c.lock)

if sg := c.recvq.dequeue(); sg != nil {
    // Found a waiting receiver. We pass the value we want to send
    // directly to the receiver, bypassing the channel buffer (if any).
    send(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true
}

// send processes a send operation on an empty channel c.
// The value ep sent by the sender is copied to the receiver sg.
// The receiver is then woken up to go on its merry way.
// Channel c must be empty and locked.  send unlocks c with unlockf.
// sg must already be dequeued from c.
// ep must be non-nil and point to the heap or the caller's stack.
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    ... 
    if sg.elem != nil {
        sendDirect(c.elemtype, sg, ep)
        sg.elem = nil
    }
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    goready(gp, skip+1)
}
```

第二种情况比较简单。通过比较 qcount 和 dataqsiz 来判断 hchan.buf 是否还有可用空间。除此之后还需要调整一下 sendx 和 qcount。

```go
lock(&c.lock)

if c.qcount < c.dataqsiz {
    // Space is available in the channel buffer. Enqueue the element to send.
    qp := chanbuf(c, c.sendx)
    if raceenabled {
        raceacquire(qp)
        racerelease(qp)
    }
    typedmemmove(c.elemtype, qp, ep)
    c.sendx++
    if c.sendx == c.dataqsiz {
        c.sendx = 0
    }
    c.qcount++
    unlock(&c.lock)
    return true
}
```

第三种情况如下。

```go
// Block on the channel. Some receiver will complete our operation for us.
gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
if t0 != 0 {
    mysg.releasetime = -1
}

mysg.elem = ep          // 一些初始化工作
mysg.waitlink = nil
mysg.g = gp
mysg.selectdone = nil
mysg.c = c
gp.waiting = mysg
gp.param = nil
c.sendq.enqueue(mysg)   // 当前 goroutine 如等待队列
goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)   //休眠
```

### 4. recv

读取 channel （ <-c ）和发送的情况非常类似。

#### 4.1 nil channel

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    if c == nil {
        if !block {
            return
        }
        gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
        throw("unreachable")
    }
    ...
}
```

#### 4.2 closed channel

从 closed channel 接收数据，如果 channel 中还有数据，接着走下面的流程。如果已经没有数据了，则返回默认值。使用 ok-idiom 方式读取的时候，第二个参数返回 false。

```go
lock(&c.lock)

if c.closed != 0 && c.qcount == 0 {
    if raceenabled {
        raceacquire(unsafe.Pointer(c))
    }
    unlock(&c.lock)
    if ep != nil {
        typedmemclr(c.elemtype, ep)
    }
    return true, false
}
```

#### 4.3 接收数据处理

当前有发送 goroutine 阻塞在 channel 上，buf 已满

```go
lock(&c.lock)

if sg := c.sendq.dequeue(); sg != nil {
    // Found a waiting sender. If buffer is size 0, receive value
    // directly from sender. Otherwise, receive from head of queue
    // and add sender's value to the tail of the queue (both map to
    // the same buffer slot because the queue is full).
    recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true, true
}
```

**buf 中有可用数据**

```go
if c.qcount > 0 {
    // Receive directly from queue
    qp := chanbuf(c, c.recvx)
    if raceenabled {
        raceacquire(qp)
        racerelease(qp)
    }
    if ep != nil {
        typedmemmove(c.elemtype, ep, qp)
    }
    typedmemclr(c.elemtype, qp)
    c.recvx++
    if c.recvx == c.dataqsiz {
        c.recvx = 0
    }
    c.qcount--
    unlock(&c.lock)
    return true, true
}
```

**buf 为空，阻塞**

```go
// no sender available: block on this channel.
gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
if t0 != 0 {
    mysg.releasetime = -1
}
// No stack splits between assigning elem and enqueuing mysg
// on gp.waiting where copystack can find it.
mysg.elem = ep
mysg.waitlink = nil
gp.waiting = mysg
mysg.g = gp
mysg.selectdone = nil
mysg.c = c
gp.param = nil
c.recvq.enqueue(mysg)
goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

```

### 5. close

关闭 channel 也就是 close(ch) 对应的代码如下（去掉部分冗余代码）。

```go
func closechan(c *hchan) {
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    lock(&c.lock)
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

    c.closed = 1

    var glist *g

    // release all readers
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
        gp.schedlink.set(glist)
        glist = gp
    }

    // release all writers (they will panic)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
        gp.schedlink.set(glist)
        glist = gp
    }
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    for glist != nil {
        gp := glist
        glist = glist.schedlink.ptr()
        gp.schedlink = 0
        goready(gp, 3)
    }
}
```

close channel 的工作除了将 c.closed 设置为 1。还需要：

- 唤醒 recvq 队列里面的阻塞 goroutine
- 唤醒 sendq 队列里面的阻塞 goroutine

处理方式是分别遍历 recvq 和 sendq 队列，将所有的 goroutine 放到 glist 队列中，最后唤醒 glist 队列中的 goroutine。

### 6. select channel

golang 中的 select 语句的实现，在 runtime/select.go 文件中，这篇文章并不打算看 select 的实现。我们要看的是 select 和 channel 一起用的时候。

```go
select {
case c <- x:
    ... foo
default:
    ... bar
}
```

会被编译为

```go
if selectnbsend(c, v) {
    ... foo
} else {
    ... bar
}
```

对应 selectnbsend 函数如下

```go
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
    return chansend(c, elem, false, getcallerpc(unsafe.Pointer(&c)))
}
```

```go
select {
case v = <-c
    ... foo
default:
    ... bar
}
```

会被编译为

```go
if selectnbrecv(&v, c) {
    ... foo
} else {
    ... bar
}
```

对应 selectnbrecv 函数如下。

```go
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
    selected, _ = chanrecv(c, elem, false)
    return
}
```

```go
select {
case v, ok = <-c:
    ... foo
default:
    ... bar
}
```

会被编译为

```go
if c != nil && selectnbrecv2(&v, &ok, c) {
    ... foo
} else {
    ... bar
}
```

对应 selectnbrecv2 函数如下。

```go
func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
    // TODO(khr): just return 2 values from this function, now that it is in Go.
    selected, *received = chanrecv(c, elem, false)
    return
}
```

### 7. 总结

Golang 的 channel 实现集中在文件 runtime/chan.go 中，本身的代码不是很复杂，但是涉及到很多其他的细节，比如 gopark 等，读起来还是有点费劲的。

### 8. 参考

- Go Source Code 1.9rc1

#

    作者：legendtkl
    链接：http://legendtkl.com/2017/08/06/golang-channel-implement/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



























