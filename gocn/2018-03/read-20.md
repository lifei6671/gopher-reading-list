## golang中的锁源码实现：Mutex

[上一篇文章](http://legendtkl.com/2016/10/13/about-lock/) 中我提到了锁，准确地说是信号量（semaphore, mutext是semaphore的一种）的实现方式有两种：wait的时候忙等待或者阻塞自己。

```go
//忙等待
wait(S) {
    while(S<=0)
        ;   //no-op
    S--
}
//阻塞
wait(semaphore *S) {
    S->value--;
    if (S->value < 0) {
        add this process to S->list;
        block()
    }
}
```

忙等待和阻塞方式各有优劣：

- 忙等待会使CPU空转，好处是如果在当前时间片内锁被其他进程释放，当前进程直接就能拿到锁而不需要CPU进行进程调度了。适用于锁占用时间较短的情况，且不适合于单处理器。
- 阻塞不会导致CPU空转，但是进程切换也需要代价，比如上下文切换，CPU Cache Miss。

下面看一下golang的源码里面是怎么实现锁的。golang里面的锁有两个特性：

1. 不支持嵌套锁
2. 可以一个goroutine lock，另一个goroutine unlock

**互斥锁**

golang中的互斥锁定义在`src/sync/mutex.go`

```go
// A Mutex is a mutual exclusion lock.
// Mutexes can be created as part of other structures;
// the zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
    state int32
    sema  uint32
}

const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexWaiterShift = iota
)
```

看上去也是使用信号量的方式来实现的。sema就是信号量，一个非负数；state表示Mutex的状态。mutexLocked表示锁是否可用（0可用，1被别的goroutine占用），mutexWoken=2表示mutex是否被唤醒，mutexWaiterShift=2表示统计阻塞在该mutex上的goroutine数目需要移位的数值。将3个常量映射到state上就是

```shell
state:   |32|31|...|3|2|1|
         \__________/ | |
               |      | |
               |      | mutex的占用状态（1被占用，0可用）
               |      |
               |      mutex的当前goroutine是否被唤醒
               |
               当前阻塞在mutex上的goroutine数
```


### 1.Lock

下面看一下mutex的lock。

```go
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }

    awoke := false
    iter := 0
    for {
        old := m.state
        new := old | mutexLocked
        if old&mutexLocked != 0 {
            if runtime_canSpin(iter) {
                // Active spinning makes sense.
                // Try to set mutexWoken flag to inform Unlock
                // to not wake other blocked goroutines.
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true
                }
                runtime_doSpin()
                iter++
                continue
            }
            new = old + 1<<mutexWaiterShift
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                panic("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&mutexLocked == 0 {
                break
            }
            runtime_Semacquire(&m.sema)
            awoke = true
            iter = 0
        }
    }

    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

这里要解释一下`atomic.CompareAndSwapInt32()`，atomic包是由golang提供的low-level的原子操作封装，主要用来解决进程同步为题，官方并不建议直接使用。我在上一篇文章中说过，操作系统级的锁的实现方案是提供原子操作，然后基本上所有锁相关都是通过这些原子操作来实现。`CompareAndSwapInt32()`就是int32型数字的compare-and-swap实现。`cas(&addr, old, new)`的意思是`if *addr==old, *addr=new`。大部分操作系统支持CAS，x86指令集上的CAS汇编指令是CMPXCHG。下面我们继续看上面的lock函数。

```go
if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
    return
}
```

首先先忽略race.Enabled相关代码，这个是go做race检测时候用的，这个时候需要带上-race，则race.Enabled被置为true。Lock函数的入口处先调用CAS尝试去获得锁，如果`m.state==0`，则将其置为1，并返回。

继续往下看，首先将m.state的值保存到old变量中，`new=old|mutexLocked`。直接看能让for退出的第三个if条件，首先调用CAS试图将m.state设置成new的值。然后看一下if里面，如果m.state之前的值也就是old如果没有被占用则表示当前goroutine拿到了锁，则break。我们先看一下new的值的变化，第一个if条件里面`new = old + 1<<mutexWaiterShift`，结合上面的mutex的state各个位的意义，这句话的意思表示mutex的等待goroutine数目加1。还有awoke为true的情况下，要将m.state的标志位取消掉，也就是这句`new &^= mutexWoken`的作用。继续看第三个if条件里面，如果里面的if判断失败，则走到`runtime_Semacquire()`。

看一下这个函数`runtime_Semacquire()`函数，由于golang1.5之后把之前C语言实现的代码都干掉了，所以现在很低层的代码都是go来实现的。通过源码中的定义我们可以知道这个其实就是信号量的wait操作：等待*s>0，然后减1。编译器里使用的是`sync_runtime.semacquire()`函数。


```go
// Semacquire waits until *s > 0 and then atomically decrements it.
// It is intended as a simple sleep primitive for use by the synchronization
// library and should not be used directly.
func runtime_Semacquire(s *uint32)

//go:linkname sync_runtime_Semacquire sync.runtime_Semacquire
func sync_runtime_Semacquire(addr *uint32) {
    semacquire(addr, true)
}

func semacquire(addr *uint32, profile bool) {
    gp := getg()
    if gp != gp.m.curg {
        throw("semacquire not on the G stack")
    }

    // Easy case.
    if cansemacquire(addr) {
        return
    }

    // Harder case:
    //  increment waiter count
    //  try cansemacquire one more time, return if succeeded
    //  enqueue itself as a waiter
    //  sleep
    //  (waiter descriptor is dequeued by signaler)
    s := acquireSudog()
    root := semroot(addr)
    t0 := int64(0)
    s.releasetime = 0
    if profile && blockprofilerate > 0 {
        t0 = cputicks()
        s.releasetime = -1
    }
    for {
        lock(&root.lock)
        // Add ourselves to nwait to disable "easy case" in semrelease.
        atomic.Xadd(&root.nwait, 1)
        // Check cansemacquire to avoid missed wakeup.
        if cansemacquire(addr) {
            atomic.Xadd(&root.nwait, -1)
            unlock(&root.lock)
            break
        }
        // Any semrelease after the cansemacquire knows we're waiting
        // (we set nwait above), so go to sleep.
        root.queue(addr, s)
        goparkunlock(&root.lock, "semacquire", traceEvGoBlockSync, 4)
        if cansemacquire(addr) {
            break
        }
    }
    if s.releasetime > 0 {
        blockevent(s.releasetime-t0, 3)
    }
    releaseSudog(s)
}
```

上面的代码有点多，我们只看和锁相关的代码。

```go
root := semroot(addr)   //seg 1

atomic.Xadd(&root.nwait, 1) // seg 2

root.queue(addr, s) //seg 3
```

seg 1代码片段`semroot()`返回结构体`semaRoot`。存储方式是先对信号量的地址做移位，然后做哈希（对251取模，这个地方为什么是左移3位和对251取模不太明白）。semaRoot相当于和mutex.sema绑定。看一下semaRoot的结构：一个sudog链表和一个nwait整型字段。nwait字段表示该信号量上等待的goroutine数目。head和tail表示链表的头和尾巴，同时为了线程安全，需要使用一个互斥量来保护链表。这个时候细心的同学应该注意到一个问题，我们前面不是从Mutex跟过来的吗，相当于Mutex的实现了使用了Mutex本身？实际上semaRoot里面的mutex只是内部使用的一个简单版本，和sync.Mutex不是同一个。现在把这些倒推回去，`runtime_Semacquire()`的作用其实就是semaphore的`wait(&s)`：如果*s<0，则将当前goroutine塞入信号量s关联的`goroutine waiting list`，并休眠。


```go
func semroot(addr *uint32) *semaRoot {
    return &semtable[(uintptr(unsafe.Pointer(addr))>>3)%semTabSize].root
}

type semaRoot struct {
    lock  mutex
    head  *sudog
    tail  *sudog
    nwait uint32 // Number of waiters. Read w/o the lock.
}

// Prime to not correlate with any user patterns.
const semTabSize = 251

var semtable [semTabSize]struct {
    root semaRoot
    pad  [sys.CacheLineSize - unsafe.Sizeof(semaRoot{})]byte
}
```

现在mutex.Lock()还剩下`runtime_canSpin(iter)`这一段，这个地方其实就是锁的自旋版本。golang对于自旋锁的取舍做了一些限制：

1. 多核; 
2. GOMAXPROCS>1; 
3. 至少有一个运行的P并且local的P队列为空。

golang的自旋尝试只会做几次，并不会一直尝试下去，感兴趣的可以跟一下源码。


```go
func sync_runtime_canSpin(i int) bool {
    // sync.Mutex is cooperative, so we are conservative with spinning.
    // Spin only few times and only if running on a multicore machine and
    // GOMAXPROCS>1 and there is at least one other running P and local runq is empty.
    // As opposed to runtime mutex we don't do passive spinning here,
    // because there can be work on global runq on on other Ps.
    if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
        return false
    }
    if p := getg().m.p.ptr(); !runqempty(p) {
        return false
    }
    return true
}

func sync_runtime_doSpin() {
    procyield(active_spin_cnt)
}
```

### 2. Unlock

Mutex的Unlock函数定义如下

```go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
    if race.Enabled {
        _ = m.state
        race.Release(unsafe.Pointer(m))
    }

    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 {
        panic("sync: unlock of unlocked mutex")
    }

    old := new
    for {
        // If there are no waiters or a goroutine has already
        // been woken or grabbed the lock, no need to wake anyone.
        if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
            return
        }
        // Grab the right to wake someone.
        new = (old - 1<<mutexWaiterShift) | mutexWoken
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            runtime_Semrelease(&m.sema)
            return
        }
        old = m.state
    }
}
```

函数入口处的四行代码和race detection相关，暂时不用管。接下来的四行代码是判断是否是嵌套锁。new是m.state-1之后的值。我们重点看for循环内部的代码。

```go
if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
    return
}
```

这两句是说：如果阻塞在该锁上的goroutine数目为0或者mutex处于lock或者唤醒状态，则返回。

```go
new = (old - 1<<mutexWaiterShift) | mutexWoken
if atomic.CompareAndSwapInt32(&m.state, old, new) {
    runtime_Semrelease(&m.sema)
    return
}
```

这里先将阻塞在mutex上的goroutine数目减一，然后将mutex置于唤醒状态。runtime_Semrelease和runtime_Semacquire的作用刚好相反，将阻塞在信号量上goroutine唤醒。有人可能会问唤醒的是哪个goroutine，那么我们可以看一下goroutine wait list的入队列和出队列代码。

```go
func (root *semaRoot) queue(addr *uint32, s *sudog) {
    s.g = getg()
    s.elem = unsafe.Pointer(addr)
    s.next = nil
    s.prev = root.tail
    if root.tail != nil {
        root.tail.next = s
    } else {
        root.head = s
    }
    root.tail = s
}

func (root *semaRoot) dequeue(s *sudog) {
    if s.next != nil {
        s.next.prev = s.prev
    } else {
        root.tail = s.prev
    }
    if s.prev != nil {
        s.prev.next = s.next
    } else {
        root.head = s.next
    }
    s.elem = nil
    s.next = nil
    s.prev = nil
}
```


如上所示，wait list入队是插在队尾，出队是从头出。

### 3. 参考

- 《Go语言学习笔记》


    作者：legendtkl
    链接：http://legendtkl.com/2016/10/23/golang-mutex/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
    






















