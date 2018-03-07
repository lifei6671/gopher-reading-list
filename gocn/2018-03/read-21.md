## 读写锁以及golang的实现

[上一篇文章](http://legendtkl.com/2016/10/23/golang-mutex/)说了golang的互斥锁（Mutex）的实现，今天来看看读写锁（RWMutex）的实现。

### 1. 读写锁是什么

下面是来自维基百科的解释

    In computer science, a readers–writer (RW) or shared-exclusive lock (also known as a multiple readers/single-writer lock[1] or multi-reader lock[2]) is a synchronization primitive that solves one of the readers–writers problems. An RW lock allows concurrent access for read-only operations, while write operations require exclusive access. This means that multiple threads can read the data in parallel but an exclusive lock is needed for writing or modifying data. When a writer is writing the data, all other writers or readers will be blocked until the writer is finished writing.

就不逐字翻译了，简明地说就是：读写锁是一种同步机制，允许多个读操作同时读取数据，但是只允许一个写操作写数据。读写锁要根据进程进入临界区的具体行为（读，写）来决定锁的占用情况。这样锁的状态就有三种了：读模式加锁、写模式加锁、无锁。

- 无锁。读/写进程都可以进入。
- 读模式锁。读进程可以进入。写进程不可以进入。
- 写模式锁。读/写进程都不可以进入。

### 2. 为什么需要读写锁

互斥锁只容许一个进程/线程处于代码临界区，而且由于锁竞争会极大地拖慢进程的执行效率。如果程序中对于共享数据的读取操作特别多，这样就很不明智了。所以对于共享数据的读取操作比较多的情况下采用读写锁。读写锁对于读操作不会导致锁竞争和进程/线程切换。

#### 2.1 读写锁的实现思路

读写锁的核心是记录reader个数。有很多种实现，下面介绍两种简单的实现。

**two mutex**

使用两个互斥锁和一个整数记录reader个数，一个互斥锁r用来保证reader个数r_cnt的更新，一个w用来保证writer互斥。伪代码如下：

```python
//读锁
RLock()
    r.lock()
    r_cnt++
    if r_cnt==1:
        w.lock()
    r.unlock()

RUnlock()
    r.lock()
    r_cnt--
    if r_cnt==0:
        w.unlock()
    r.unlock()
//写锁
Lock()
    w.lock()

Unlock()
    w.unlock()
```

**mutex and condition**

另外一种常见的实现方式是使用mutex和condition variable，思路很简单：RLock的时候如果发现有writer，则阻塞到condition variable；Lock的时候如果发现有writer，则阻塞。

```python
RLock()
    m.lock()
    while w:
        c.wait(m)
    r++
    m.unlock()

RUnlock()
    m.lock()
    r--
    if r==0:
        c.signal_all()
    m.unlock()

Lock()
    m.lock()
    while w || r > 0 :
        c.wait(m)
    w = true
    m.unlock()

Unlock()
    m.lock()
    w = false
    c.signal_all()
    m.unlock()
```

#### 2.1 golang读写锁代码实现

读写锁的定义在sync/rwmutex.go文件中。


```go
// An RWMutex is a reader/writer mutual exclusion lock.
// The lock can be held by an arbitrary number of readers
// or a single writer.
// RWMutexes can be created as part of other
// structures; the zero value for a RWMutex is
// an unlocked mutex.
type RWMutex struct {
    w           Mutex  // held if there are pending writers
    writerSem   uint32 // semaphore for writers to wait for completing readers
    readerSem   uint32 // semaphore for readers to wait for completing writers
    readerCount int32  // number of pending readers
    readerWait  int32  // number of departing readers
}
```

从定义中可以看出RWMutex内部使用了Mutex，mutex其实是用来对写操作进行互斥的。同时使用两个信号量writerSem和readerSem来处理读写进程之间的关系。最后使用readerCount记录reader的个数，使用readerWait记录writer前面的reader的个数。感觉说的有点不太清楚，还是直接看代码吧。

RWMutex struct定义了四个方法如下。

```go
// RLock locks rw for reading.
func (rw *RWMutex) RLock()

// RUnlock undoes a single RLock call;
func (rw *RWMutex) RUnlock()

// Lock locks rw for writing
func (rw *RWMutex) Lock()

// Unlock unlocks rw for writing.
func (rw *RWMutex) Unlock()
```

去掉一些golang做race detect代码，所有的代码实现如下：

```go
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // A writer is pending, wait for it.
        runtime_Semacquire(&rw.readerSem)
    }
}

func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        if r+1 == 0 || r+1 == -rwmutexMaxReaders {
            raceEnable()
            panic("sync: RUnlock of unlocked RWMutex")
        }
        // A writer is pending.
        if atomic.AddInt32(&rw.readerWait, -1) == 0 {
            // The last reader unblocks the writer.
            runtime_Semrelease(&rw.writerSem)
        }
    }
}

func (rw *RWMutex) Lock() {
    // First, resolve competition with other writers.
    rw.w.Lock()
    // Announce to readers there is a pending writer.
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // Wait for active readers.
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_Semacquire(&rw.writerSem)
    }
}

func (rw *RWMutex) Unlock() {
    // Announce to readers there is no active writer.
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    if r >= rwmutexMaxReaders {
        raceEnable()
        panic("sync: Unlock of unlocked RWMutex")
    }
    // Unblock blocked readers, if any.
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem)
    }
    // Allow other writers to proceed.
    rw.w.Unlock()
}
```

首先看一下这两个函数，分别对应我这篇文章[《当我们谈论锁，我们谈什么》](http://legendtkl.com/2016/10/13/about-lock/)中的wait和signal操作。

```go
//Semacquire waits until *s > 0 and then atomically decrements it.
func runtime_Semacquire(s *uint32)

// Semrelease atomically increments *s and notifies a waiting goroutine
func runtime_Semrelease(s *uint32)
```

首先看一下RLock()函数，它对readerCount进行加一操作（原子操作），如果readerCount<0，则wait(readerSem)。等等，为什么readerCount会小于0呢？往下看发现writer的Lock()会对readerCount做减法操作（原子操作），用来表示现在有writer。这个时候表示reader前面有writer，reader阻塞到信号量readerSem上，然后我们发现Unlock()中做了 signal(readerSem)，也就是唤醒阻塞在readerSem上的进程。同时Lock()将当前的readerCount值加到readerWait上，如果readerWait不为0，则表示这个writer前有reader，选择wait(writerSem)。类似的RUnlock()里面做了signal(writerSem)。稍微整理一下信号量的wait和signal。

```python
//reader到来，前面有writer
RLock()
    if writer exist:
        wait(readerSem)

Unlock()
    signal(readerSem)

//writer到来，前面有reader
Lock()
    if reader exist:
        wait(writerSem)

RUnlock()
    signal(writerSem)
```

不得不说golang的读写锁实现还是挺巧妙的。

### 3. 参考：

- [ikipedia](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock#cite_note-pthreads-7)
- [stackoverflow](http://stackoverflow.com/questions/12033188/how-would-a-readers-writer-lock-be-implemented-in-c11)

#

    作者：legendtkl
    链接：http://legendtkl.com/2016/10/26/rwmutex/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。




















