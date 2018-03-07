## Golang 内存管理

Golang 的内存管理基于 tcmalloc，可以说起点挺高的。但是 Golang 在实现的时候还做了很多优化，我们下面通过源码来看一下 Golang 的内存管理实现。下面的源码分析基于 go1.8rc3。

### 1.tcmalloc 介绍

关于 tcmalloc 可以参考这篇文章 tcmalloc 介绍，原始论文可以参考 TCMalloc : Thread-Caching Malloc。

### 2. Golang 内存管理

#### 0. 准备知识

这里先简单介绍一下 Golang 运行调度。在 Golang 里面有三个基本的概念：G, M, P。

- G: Goroutine 执行的上下文环境。
- M: 操作系统线程。
- P: Processer。进程调度的关键，调度器，也可以认为约等于 CPU。

一个 Goroutine 的运行需要 G + P + M 三部分结合起来。好，先简单介绍到这里，更详细的放在后面的文章里面来说。

#### 1. 逃逸分析（escape analysis）

对于手动管理内存的语言，比如 C/C++，我们使用 malloc 或者 new 申请的变量会被分配到堆上。但是 Golang 并不是这样，虽然 Golang 语言里面也有 new。Golang 编译器决定变量应该分配到什么地方时会进行逃逸分析。下面是一个简单的例子。


```go
package main

import ()

func foo() *int {
    var x int
    return &x
}

func bar() int {
    x := new(int)
    *x = 1
    return *x
}

func main() {

}
```

将上面文件保存为 escape.go，执行下面命令

```go
$ go run -gcflags '-m -l' escape.go
./escape.go:6: moved to heap: x
./escape.go:7: &x escape to heap
./escape.go:11: bar new(int) does not escape

```

上面的意思是 foo() 中的 x 最后在堆上分配，而 bar() 中的 x 最后分配在了栈上。在官网 (golang.org) FAQ 上有一个关于变量分配的问题如下：

**How do I know whether a variable is allocated on the heap or the stack?**

    From a correctness standpoint, you don’t need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.
    
    The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function’s stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
    
    In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

简单翻译一下。

**如何得知变量是分配在栈（stack）上还是堆（heap）上？**

    准确地说，你并不需要知道。Golang 中的变量只要被引用就一直会存活，存储在堆上还是栈上由内部实现决定而和具体的语法没有关系。
    
    知道变量的存储位置确实和效率编程有关系。如果可能，Golang 编译器会将函数的局部变量分配到函数栈帧（stack frame）上。然而，如果编译器不能确保变量在函数 return 之后不再被引用，编译器就会将变量分配到堆上。而且，如果一个局部变量非常大，那么它也应该被分配到堆上而不是栈上。
    
    当前情况下，如果一个变量被取地址，那么它就有可能被分配到堆上。然而，还要对这些变量做逃逸分析，如果函数 return 之后，变量不再被引用，则将其分配到栈上。


### 2. 关键数据结构

几个关键的地方：

- mcache: per-P cache，可以认为是 local cache。
- mcentral: 全局 cache，mcache 不够用的时候向 mcentral 申请。
- mheap: 当 mcentral 也不够用的时候，通过 mheap 向操作系统申请。

可以将其看成多级内存分配器。

#### 2.1 mcache

我们知道每个 Gorontine 的运行都是绑定到一个 P 上面，mcache 是每个 P 的 cache。这么做的好处是分配内存时不需要加锁。mcache 结构如下。

```go
// Per-thread (in Go, per-P) cache for small objects.
// No locking needed because it is per-thread (per-P).
type mcache struct {
    // The following members are accessed on every malloc,
    // so they are grouped here for better caching.
    next_sample int32   // trigger heap sample after allocating this many bytes
    local_scan  uintptr // bytes of scannable heap allocated

    // 小对象分配器，小于 16 byte 的小对象都会通过 tiny 来分配。
    tiny             uintptr
    tinyoffset       uintptr
    local_tinyallocs uintptr // number of tiny allocs not counted in other stats

    // The rest is not accessed on every malloc.
    alloc [_NumSizeClasses]*mspan // spans to allocate from

    stackcache [_NumStackOrders]stackfreelist

    // Local allocator stats, flushed during GC.
    local_nlookup    uintptr                  // number of pointer lookups
    local_largefree  uintptr                  // bytes freed for large objects (>maxsmallsize)
    local_nlargefree uintptr                  // number of frees for large objects (>maxsmallsize)
    local_nsmallfree [_NumSizeClasses]uintptr // number of frees for small objects (<=maxsmallsize)
}
```

我们可以暂时只关注 `alloc [_NumSizeClasses]*mspan`，这是一个大小为 67 的指针（指针指向 mspan ）数组（_NumSizeClasses = 67），每个数组元素用来包含特定大小的块。当要分配内存大小时，为 object 在 alloc 数组中选择合适的元素来分配。67 种块大小为 0，8 byte, 16 byte, … ，这个和 tcmalloc 稍有区别。


```go
//file: sizeclasses.go
var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}
```

这里仔细想有个小问题，上面的 alloc 类似内存池的 freelist 数组或者链表，正常实现每个数组元素是一个链表，链表由特定大小的块串起来。但是这里统一使用了 mspan 结构，那么只有一种可能，就是 mspan 中记录了需要分配的块大小。我们来看一下 mspan 的结构。

#### 2.2 mspan

span 在 tcmalloc 中作为一种管理内存的基本单位而存在。Golang 的 mspan 的结构如下，省略了部分内容。

```go
type mspan struct {
    next *mspan     // next span in list, or nil if none
    prev *mspan     // previous span in list, or nil if none
    list *mSpanList // For debugging. TODO: Remove.

    startAddr     uintptr   // address of first byte of span aka s.base()
    npages        uintptr   // number of pages in span
    stackfreelist gclinkptr // list of free stacks, avoids overloading freelist
    // freeindex is the slot index between 0 and nelems at which to begin scanning
    // for the next free object in this span.
    freeindex uintptr
    // TODO: Look up nelems from sizeclass and remove this field if it
    // helps performance.
    nelems uintptr // number of object in the span.
    ...
    // 用位图来管理可用的 free object，1 表示可用
    allocCache uint64
    
    ...
    sizeclass   uint8      // size class
    ...
    elemsize    uintptr    // computed from sizeclass or from npages
    ...
}
```

从上面的结构可以看出：

- next, prev: 指针域，因为 mspan 一般都是以链表形式使用。
- npages: mspan 的大小为 page 大小的整数倍。
- sizeclass: 0 ~ _NumSizeClasses 之间的一个值，这个解释了我们的疑问。比如，sizeclass = 3，那么这个 mspan 被分割成 32 byte 的块。
- elemsize: 通过 sizeclass 或者 npages 可以计算出来。比如 sizeclass = 3, elemsize = 32 byte。对于大于 32Kb 的内存分配，都是分配整数页，elemsize = page_size * npages。
- nelems: span 中包块的总数目。
- freeindex: 0 ~ nelemes-1，表示分配到第几个块。

#### 2.3 mcentral

上面说到当 mcache 不够用的时候，会从 mcentral 申请。那我们下面就来介绍一下 mcentral。

```go
type mcentral struct {
    lock      mutex
    sizeclass int32
    nonempty  mSpanList // list of spans with a free object, ie a nonempty free list
    empty     mSpanList // list of spans with no free objects (or cached in an mcache)
}

type mSpanList struct {
    first *mspan
    last  *mspan
}
```

mcentral 分析：

- sizeclass: 也有成员 sizeclass，那么 mcentral 是不是也有 67 个呢？是的。
- lock: 因为会有多个 P 过来竞争。
- nonempty: mspan 的双向链表，当前 mcentral 中可用的 mspan list。
- empty: 已经被使用的，可以认为是一种对所有 mspan 的 track。

问题来了，mcentral 存在于什么地方？虽然在上面我们将 mcentral 和 mheap 作为两个部分来讲，但是作为全局的结构，这两部分是可以定义在一起的。实际上也是这样，mcentral 包含在 mheap 中。

#### 2.4 mheap

Golang 中的 mheap 结构定义如下。

```go
type mheap struct {
    lock      mutex
    free      [_MaxMHeapList]mSpanList // free lists of given length
    freelarge mSpanList                // free lists length >= _MaxMHeapList
    busy      [_MaxMHeapList]mSpanList // busy lists of large objects of given length
    busylarge mSpanList                // busy lists of large objects length >= _MaxMHeapList
    sweepgen  uint32                   // sweep generation, see comment in mspan
    sweepdone uint32                   // all spans are swept

    // allspans is a slice of all mspans ever created. Each mspan
    // appears exactly once.
    //
    // The memory for allspans is manually managed and can be
    // reallocated and move as the heap grows.
    //
    // In general, allspans is protected by mheap_.lock, which
    // prevents concurrent access as well as freeing the backing
    // store. Accesses during STW might not hold the lock, but
    // must ensure that allocation cannot happen around the
    // access (since that may free the backing store).
    allspans []*mspan // all spans out there

    // spans is a lookup table to map virtual address page IDs to *mspan.
    // For allocated spans, their pages map to the span itself.
    // For free spans, only the lowest and highest pages map to the span itself.
    // Internal pages map to an arbitrary span.
    // For pages that have never been allocated, spans entries are nil.
    //
    // This is backed by a reserved region of the address space so
    // it can grow without moving. The memory up to len(spans) is
    // mapped. cap(spans) indicates the total reserved memory.
    spans []*mspan

    // sweepSpans contains two mspan stacks: one of swept in-use
    // spans, and one of unswept in-use spans. These two trade
    // roles on each GC cycle. Since the sweepgen increases by 2
    // on each cycle, this means the swept spans are in
    // sweepSpans[sweepgen/2%2] and the unswept spans are in
    // sweepSpans[1-sweepgen/2%2]. Sweeping pops spans from the
    // unswept stack and pushes spans that are still in-use on the
    // swept stack. Likewise, allocating an in-use span pushes it
    // on the swept stack.
    sweepSpans [2]gcSweepBuf

    _ uint32 // align uint64 fields on 32-bit for atomics

    // Proportional sweep
    pagesInUse        uint64  // pages of spans in stats _MSpanInUse; R/W with mheap.lock
    spanBytesAlloc    uint64  // bytes of spans allocated this cycle; updated atomically
    pagesSwept        uint64  // pages swept this cycle; updated atomically
    sweepPagesPerByte float64 // proportional sweep ratio; written with lock, read without
    // TODO(austin): pagesInUse should be a uintptr, but the 386
    // compiler can't 8-byte align fields.

    // Malloc stats.
    largefree  uint64                  // bytes freed for large objects (>maxsmallsize)
    nlargefree uint64                  // number of frees for large objects (>maxsmallsize)
    nsmallfree [_NumSizeClasses]uint64 // number of frees for small objects (<=maxsmallsize)

    // range of addresses we might see in the heap
    bitmap         uintptr // Points to one byte past the end of the bitmap
    bitmap_mapped  uintptr
    arena_start    uintptr
    arena_used     uintptr // always mHeap_Map{Bits,Spans} before updating
    arena_end      uintptr
    arena_reserved bool

    // central free lists for small size classes.
    // the padding makes sure that the MCentrals are
    // spaced CacheLineSize bytes apart, so that each MCentral.lock
    // gets its own cache line.
    central [_NumSizeClasses]struct {
        mcentral mcentral
        pad      [sys.CacheLineSize]byte
    }

    spanalloc             fixalloc // allocator for span*
    cachealloc            fixalloc // allocator for mcache*
    specialfinalizeralloc fixalloc // allocator for specialfinalizer*
    specialprofilealloc   fixalloc // allocator for specialprofile*
    speciallock           mutex    // lock for special record allocators.
}

var mheap_ mheap
```

mheap_ 是一个全局变量，会在系统初始化的时候初始化（在函数 mallocinit() 中）。我们先看一下 mheap 具体结构。

- allspans []*mspan: 所有的 spans 都是通过 mheap_ 申请，所有申请过的 mspan 都会记录在 allspans。结构体中的 lock 就是用来保证并发安全的。注释中有关于 STW 的说明，这个之后会在 Golang 的 GC 文章中细说。
- central [_NumSizeClasses]…: 这个就是之前介绍的 mcentral ，每种大小的块对应一个 mcentral。mcentral 上面介绍过了。pad 可以认为是一个字节填充，为了避免伪共享（false sharing）问题的。False Sharing 可以参考 False Sharing - wikipedia，这里就不细说了。
- sweepgen, sweepdone: GC 相关。（Golang 的 GC 策略是 Mark & Sweep, 这里是用来表示 sweep 的，这里就不再深入了。）
- free [_MaxMHeapList]mSpanList: 这是一个 SpanList 数组，每个 SpanList 里面的 mspan 由 1 ~ 127 (_MaxMHeapList - 1) 个 page 组成。比如 free[3] 是由包含 3 个 page 的 mspan 组成的链表。free 表示的是 free list，也就是未分配的。对应的还有 busy list。
- freelarge mSpanList: mspan 组成的链表，每个元素（也就是 mspan）的 page 个数大于 127。对应的还有 busylarge。
- spans []*mspan: 记录 arena 区域页号（page number）和 mspan 的映射关系。
- arena_start, arena_end, arena_used: 要解释这几个变量之前要解释一下 arena。arena 是 Golang 中用于分配内存的连续虚拟地址区域。由 mheap 管理，堆上申请的所有内存都来自 arena。那么如何标志内存可用呢？操作系统的常见做法用两种：一种是用链表将所有的可用内存都串起来；另一种是使用位图来标志内存块是否可用。结合上面一条 spans，内存的布局是下面这样的。
```
    +-----------------------+---------------------+-----------------------+
    |    spans              |    bitmap           |   arena               |
    +-----------------------+---------------------+-----------------------+
```
- spanalloc, cachealloc fixalloc: fixalloc 是 free-list，用来分配特定大小的块。
- 剩下的是一些统计信息和 GC 相关的信息，这里暂且按住不表，以后专门拿出来说。

### 3. 初始化

在系统初始化阶段，上面介绍的几个结构会被进行初始化，我们直接看一下初始化代码：mallocinit()。

```go
func mallocinit() {
    //一些系统检测代码，略去
    var p, bitmapSize, spansSize, pSize, limit uintptr
    var reserved bool

    // limit = runtime.memlimit();
    // See https://golang.org/issue/5049
    // TODO(rsc): Fix after 1.1.
    limit = 0
  
    //系统指针大小 PtrSize = 8，表示这是一个 64 位系统。
    if sys.PtrSize == 8 && (limit == 0 || limit > 1<<30) {
        //这里的 arenaSize, bitmapSize, spansSize 分别对应 mheap 那一小节里面提到 arena 区大小，bitmap 区大小，spans 区大小。
        arenaSize := round(_MaxMem, _PageSize)
        bitmapSize = arenaSize / (sys.PtrSize * 8 / 2)
        spansSize = arenaSize / _PageSize * sys.PtrSize
        spansSize = round(spansSize, _PageSize)
        //尝试从不同地址开始申请
        for i := 0; i <= 0x7f; i++ {
            switch {
            case GOARCH == "arm64" && GOOS == "darwin":
                p = uintptr(i)<<40 | uintptrMask&(0x0013<<28)
            case GOARCH == "arm64":
                p = uintptr(i)<<40 | uintptrMask&(0x0040<<32)
            default:
                p = uintptr(i)<<40 | uintptrMask&(0x00c0<<32)
            }
            pSize = bitmapSize + spansSize + arenaSize + _PageSize
            //向 OS 申请大小为 pSize 的连续的虚拟地址空间
            p = uintptr(sysReserve(unsafe.Pointer(p), pSize, &reserved))
            if p != 0 {
                break
            }
        }
    }
    //这里是 32 位系统代码对应的操作，略去。
    ...
    
    p1 := round(p, _PageSize)

    spansStart := p1
    mheap_.bitmap = p1 + spansSize + bitmapSize
    if sys.PtrSize == 4 {
        // Set arena_start such that we can accept memory
        // reservations located anywhere in the 4GB virtual space.
        mheap_.arena_start = 0
    } else {
        mheap_.arena_start = p1 + (spansSize + bitmapSize)
    }
    mheap_.arena_end = p + pSize
    mheap_.arena_used = p1 + (spansSize + bitmapSize)
    mheap_.arena_reserved = reserved

    if mheap_.arena_start&(_PageSize-1) != 0 {
        println("bad pagesize", hex(p), hex(p1), hex(spansSize), hex(bitmapSize), hex(_PageSize), "start", hex(mheap_.arena_start))
        throw("misrounded allocation in mallocinit")
    }

    // Initialize the rest of the allocator.
    mheap_.init(spansStart, spansSize)
    _g_ := getg()
    _g_.m.mcache = allocmcache()
}
```

上面对代码做了简单的注释，下面详细解说其中的部分功能函数。

#### 3.1 arena 相关

arena 相关地址的大小初始化代码如下。

```go
arenaSize := round(_MaxMem, _PageSize)
bitmapSize = arenaSize / (sys.PtrSize * 8 / 2)
spansSize = arenaSize / _PageSize * sys.PtrSize
spansSize = round(spansSize, _PageSize)

_MaxMem = uintptr(1<<_MHeapMap_TotalBits - 1)

```

首先解释一下变量 _MaxMem ，里面还有一个变量就不再列出来了。简单来说 _MaxMem 就是系统为 arena 区分配的大小：64 位系统分配 512 G；对于 Windows 64 位系统，arena 区分配 32 G。round 是一个对齐操作，向上取 _PageSize 的倍数。实现也很有意思，代码如下。

```go
// round n up to a multiple of a.  a must be a power of 2.
func round(n, a uintptr) uintptr {
    return (n + a - 1) &^ (a - 1)
}
```

bitmap 用两个 bit 表示一个字的可用状态，那么算下来 bitmap 的大小为 16 G。读过 Golang 源码的同学会发现其实这段代码的注释里写的 bitmap 的大小为 32 G。其实是这段注释很久没有更新了，之前是用 4 个 bit 来表示一个字的可用状态，这真是一个悲伤的故事啊。

spans 记录的 arena 区的块页号和对应的 mspan 指针的对应关系。比如 arena 区内存地址 x，对应的页号就是 `page_num = (x - arena_start) / page_size`，那么 spans 就会记录 spans[page_num] = x。如果 arena 为 512 G的话，spans 区的大小为 512 G / 8K * 8 = 512 M。这里值得注意的是 Golang 的内存管理虚拟地址页大小为 8k。

```go
_PageSize = 1 << _PageShift

_PageShift = 13
```

所以这一段连续的的虚拟内存布局（64 位）如下：

```
+-----------------------+---------------------+-----------------------+
|    spans 512M         |    bitmap 16G       |   arena 512           |
+-----------------------+---------------------+-----------------------+

```

#### 3.2 虚拟地址申请

主要是下面这段代码。

```go
//尝试从不同地址开始申请
for i := 0; i <= 0x7f; i++ {
    switch {
    case GOARCH == "arm64" && GOOS == "darwin":
        p = uintptr(i)<<40 | uintptrMask&(0x0013<<28)
    case GOARCH == "arm64":
        p = uintptr(i)<<40 | uintptrMask&(0x0040<<32)
    default:
        p = uintptr(i)<<40 | uintptrMask&(0x00c0<<32)
    }
    pSize = bitmapSize + spansSize + arenaSize + _PageSize
    //向 OS 申请大小为 pSize 的连续的虚拟地址空间
    p = uintptr(sysReserve(unsafe.Pointer(p), pSize, &reserved))
    if p != 0 {
        break
    }
}
```

初始化的时候，Golang 向操作系统申请一段连续的地址空间，就是上面的 spans + bitmap + arena。p 就是这段连续地址空间的开始地址，不同平台的 p 取值不一样。像 OS 申请的时候视不同的 OS 版本，调用不同的系统调用，比如 Unix 系统调用 mmap (mmap 想操作系统内核申请新的虚拟地址区间，可指定起始地址和长度)，Windows 系统调用 VirtualAlloc （类似 mmap）。

```go
//bsd
func sysReserve(v unsafe.Pointer, n uintptr, reserved *bool) unsafe.Pointer {
    if sys.PtrSize == 8 && uint64(n) > 1<<32 || sys.GoosNacl != 0 {
        *reserved = false
        return v
    }

    p := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
    if uintptr(p) < 4096 {
        return nil
    }
    *reserved = true
    return p
}

//darwin
func sysReserve(v unsafe.Pointer, n uintptr, reserved *bool) unsafe.Pointer {
    *reserved = true
    p := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
    if uintptr(p) < 4096 {
        return nil
    }
    return p
}

//linux
func sysReserve(v unsafe.Pointer, n uintptr, reserved *bool) unsafe.Pointer {
    ...
    p := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
    if uintptr(p) < 4096 {
        return nil
    }
    *reserved = true
    return p
}
//windows
func sysReserve(v unsafe.Pointer, n uintptr, reserved *bool) unsafe.Pointer {
    *reserved = true
    // v is just a hint.
    // First try at v.
    v = unsafe.Pointer(stdcall4(_VirtualAlloc, uintptr(v), n, _MEM_RESERVE, _PAGE_READWRITE))
    if v != nil {
        return v
    }

    // Next let the kernel choose the address.
    return unsafe.Pointer(stdcall4(_VirtualAlloc, 0, n, _MEM_RESERVE, _PAGE_READWRITE))
}
```

#### 3.3 mheap 初始化

我们上面介绍 mheap 结构的时候知道 spans, bitmap, arena 都是存在于 mheap 中的，从操作系统申请完地址之后就是初始化 mheap 了。

```go
func mallocinit() {
    ...
    p1 := round(p, _PageSize)

    spansStart := p1
    mheap_.bitmap = p1 + spansSize + bitmapSize
    if sys.PtrSize == 4 {
        // Set arena_start such that we can accept memory
        // reservations located anywhere in the 4GB virtual space.
        mheap_.arena_start = 0
    } else {
        mheap_.arena_start = p1 + (spansSize + bitmapSize)
    }
    mheap_.arena_end = p + pSize
    mheap_.arena_used = p1 + (spansSize + bitmapSize)
    mheap_.arena_reserved = reserved

    if mheap_.arena_start&(_PageSize-1) != 0 {
        println("bad pagesize", hex(p), hex(p1), hex(spansSize), hex(bitmapSize), hex(_PageSize), "start", hex(mheap_.arena_start))
        throw("misrounded allocation in mallocinit")
    }

    // Initialize the rest of the allocator.
    mheap_.init(spansStart, spansSize)
    //获取当前 G
    _g_ := getg()
    // 获取 G 上绑定的 M 的 mcache
    _g_.m.mcache = allocmcache()
}
```

p 是从连续虚拟地址的起始地址，先进行对齐，然后初始化 arena，bitmap，spans 地址。mheap_.init()会初始化 fixalloc 等相关的成员，还有 mcentral 的初始化。

```go
func (h *mheap) init(spansStart, spansBytes uintptr) {
    h.spanalloc.init(unsafe.Sizeof(mspan{}), recordspan, unsafe.Pointer(h), &memstats.mspan_sys)
    h.cachealloc.init(unsafe.Sizeof(mcache{}), nil, nil, &memstats.mcache_sys)
    h.specialfinalizeralloc.init(unsafe.Sizeof(specialfinalizer{}), nil, nil, &memstats.other_sys)
    h.specialprofilealloc.init(unsafe.Sizeof(specialprofile{}), nil, nil, &memstats.other_sys)

    h.spanalloc.zero = false

    // h->mapcache needs no init
    for i := range h.free {
        h.free[i].init()
        h.busy[i].init()
    }

    h.freelarge.init()
    h.busylarge.init()
    for i := range h.central {
        h.central[i].mcentral.init(int32(i))
    }

    sp := (*slice)(unsafe.Pointer(&h.spans))
    sp.array = unsafe.Pointer(spansStart)
    sp.len = 0
    sp.cap = int(spansBytes / sys.PtrSize)
}
```

mheap 初始化之后，对当前的线程也就是 M 进行初始化。

```go
//获取当前 G
_g_ := getg()
// 获取 G 上绑定的 M 的 mcache
_g_.m.mcache = allocmcache()

```

#### 3.4 per-P mcache 初始化

上面好像并没有说到针对 P 的 mcache 初始化，因为这个时候还没有初始化 P。我们看一下 bootstrap 的代码。

```go
func schedinit() {
    ...
    mallocinit()
    ...
    
    if procs > _MaxGomaxprocs {
        procs = _MaxGomaxprocs
    }
    if procresize(procs) != nil {
        ...
    }
}
```

其中 mallocinit() 上面说过了。对 P 的初始化在函数 procresize() 中执行，我们下面只看内存相关的部分。

```go
func procresize(nprocs int32) *p {
    ...
    // initialize new P's
    for i := int32(0); i < nprocs; i++ {
        pp := allp[i]
        if pp == nil {
            pp = new(p)
            pp.id = i
            pp.status = _Pgcstop
            pp.sudogcache = pp.sudogbuf[:0]
            for i := range pp.deferpool {
                pp.deferpool[i] = pp.deferpoolbuf[i][:0]
            }
            atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
        }
        // P mcache 初始化
        if pp.mcache == nil {
            if old == 0 && i == 0 {
                if getg().m.mcache == nil {
                    throw("missing mcache?")
                }
                // P[0] 分配给主 Goroutine
                pp.mcache = getg().m.mcache // bootstrap
            } else {
                // P[0] 之外的 P 申请 mcache
                pp.mcache = allocmcache()
            }
        }
        ...
    }
    ...
}
```

所有的 P 都存放在一个全局数组 allp 中，procresize() 的目的就是将 allp 中用到的 P 进行初始化，同时对多余 P 的资源剥离。

### 4. 内存分配

先说一下给对象 object 分配内存的主要流程：

1. object size > 32K，则使用 mheap 直接分配。
2. object size < 16 byte，使用 mcache 的小对象分配器 tiny 直接分配。 （其实 tiny 就是一个指针，暂且这么说吧。）
3. object size > 16 byte && size <=32K byte 时，先使用 mcache 中对应的 size class 分配。
4. 如果 mcache 对应的 size class 的 span 已经没有可用的块，则向 mcentral 请求。
5. 如果 mcentral 也没有可用的块，则向 mheap 申请，并切分。
6. 如果 mheap 也没有合适的 span，则想操作系统申请。

我们看一下在堆上，也就是 arena 区分配内存的相关函数。

```go
package main

func foo() *int {
    x := 1
    return &x
}

func main() {
    x := foo()
    println(*x)
}
```

根据之前介绍的逃逸分析，foo() 中的 x 会被分配到堆上。把上面代码保存为 test1.go 看一下汇编代码。

```shell
$ go build -gcflags '-l' -o test1 test1.go
$ go tool objdump -s "main\.foo" test1
TEXT main.foo(SB) /Users/didi/code/go/malloc_example/test2.go
    test2.go:3  0x2040  65488b0c25a0080000  GS MOVQ GS:0x8a0, CX
    test2.go:3  0x2049  483b6110        CMPQ 0x10(CX), SP
    test2.go:3  0x204d  762a            JBE 0x2079
    test2.go:3  0x204f  4883ec10        SUBQ $0x10, SP
    test2.go:4  0x2053  488d1d66460500      LEAQ 0x54666(IP), BX
    test2.go:4  0x205a  48891c24        MOVQ BX, 0(SP)
    test2.go:4  0x205e  e82d8f0000      CALL runtime.newobject(SB)
    test2.go:4  0x2063  488b442408      MOVQ 0x8(SP), AX
    test2.go:4  0x2068  48c70001000000      MOVQ $0x1, 0(AX)
    test2.go:5  0x206f  4889442418      MOVQ AX, 0x18(SP)
    test2.go:5  0x2074  4883c410        ADDQ $0x10, SP
    test2.go:5  0x2078  c3          RET
    test2.go:3  0x2079  e8a28d0400      CALL runtime.morestack_noctxt(SB)
    test2.go:3  0x207e  ebc0            JMP main.foo(SB)
```

堆上内存分配调用了 runtime 包的 newobject 函数。

```go
func newobject(typ *_type) unsafe.Pointer {
    return mallocgc(typ.size, typ, true)
}

func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    ... 
    c := gomcache()
    var x unsafe.Pointer
    noscan := typ == nil || typ.kind&kindNoPointers != 0
    if size <= maxSmallSize {
        // object size <= 32K
        if noscan && size < maxTinySize {
            // 小于 16 byte 的小对象分配
            off := c.tinyoffset
            // Align tiny pointer for required (conservative) alignment.
            if size&7 == 0 {
                off = round(off, 8)
            } else if size&3 == 0 {
                off = round(off, 4)
            } else if size&1 == 0 {
                off = round(off, 2)
            }
            if off+size <= maxTinySize && c.tiny != 0 {
                // The object fits into existing tiny block.
                x = unsafe.Pointer(c.tiny + off)
                c.tinyoffset = off + size
                c.local_tinyallocs++
                mp.mallocing = 0
                releasem(mp)
                return x
            }
            // Allocate a new maxTinySize block.
            span := c.alloc[tinySizeClass]
            v := nextFreeFast(span)
            if v == 0 {
                v, _, shouldhelpgc = c.nextFree(tinySizeClass)
            }
            x = unsafe.Pointer(v)
            (*[2]uint64)(x)[0] = 0
            (*[2]uint64)(x)[1] = 0
            // See if we need to replace the existing tiny block with the new one
            // based on amount of remaining free space.
            if size < c.tinyoffset || c.tiny == 0 {
                c.tiny = uintptr(x)
                c.tinyoffset = size
            }
            size = maxTinySize
        } else {
            // object size >= 16 byte  && object size <= 32K byte
            var sizeclass uint8
            if size <= smallSizeMax-8 {
                sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
            } else {
                sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
            }
            size = uintptr(class_to_size[sizeclass])
            span := c.alloc[sizeclass]
            v := nextFreeFast(span)
            if v == 0 {
                v, span, shouldhelpgc = c.nextFree(sizeclass)
            }
            x = unsafe.Pointer(v)
            if needzero && span.needzero != 0 {
                memclrNoHeapPointers(unsafe.Pointer(v), size)
            }
        }
    } else {
        //object size > 32K byte
        var s *mspan
        shouldhelpgc = true
        systemstack(func() {
            s = largeAlloc(size, needzero)
        })
        s.freeindex = 1
        s.allocCount = 1
        x = unsafe.Pointer(s.base())
        size = s.elemsize
    }
}
```

整个分配过程可以根据 object size 拆解成三部分：size < 16 byte, 16 byte <= size <= 32 K byte, size > 32 K byte。

#### 4.1 size 小于 16 byte

对于小于 16 byte 的内存块，mcache 有个专门的内存区域 tiny 用来分配，tiny 是指针，指向开始地址。

```go
func mallocgc(...) {
    ...
            off := c.tinyoffset
            // 地址对齐
            if size&7 == 0 {
                off = round(off, 8)
            } else if size&3 == 0 {
                off = round(off, 4)
            } else if size&1 == 0 {
                off = round(off, 2)
            }
            // 分配
            if off+size <= maxTinySize && c.tiny != 0 {
                // The object fits into existing tiny block.
                x = unsafe.Pointer(c.tiny + off)
                c.tinyoffset = off + size
                c.local_tinyallocs++
                mp.mallocing = 0
                releasem(mp)
                return x
            }
            // tiny 不够了，为其重新分配一个 16 byte 内存块
            span := c.alloc[tinySizeClass]
            v := nextFreeFast(span)
            if v == 0 {
                v, _, shouldhelpgc = c.nextFree(tinySizeClass)
            }
            x = unsafe.Pointer(v)
            //将申请的内存块全置为 0
            (*[2]uint64)(x)[0] = 0
            (*[2]uint64)(x)[1] = 0
            // See if we need to replace the existing tiny block with the new one
            // based on amount of remaining free space.
            // 如果申请的内存块用不完，则将剩下的给 tiny，用 tinyoffset 记录分配了多少。
            if size < c.tinyoffset || c.tiny == 0 {
                c.tiny = uintptr(x)
                c.tinyoffset = size
            }
            size = maxTinySize
}
```

如上所示，tinyoffset 表示 tiny 当前分配到什么地址了，之后的分配根据 tinyoffset 寻址。先根据要分配的对象大小进行地址对齐，比如 size 是 8 的倍数，tinyoffset 和 8 对齐。然后就是进行分配。如果 tiny 剩余的空间不够用，则重新申请一个 16 byte 的内存块，并分配给 object。如果有结余，则记录在 tiny 上。

#### 4.2 size 大于 32 K byte

对于大于 32 Kb 的内存分配，直接跳过 mcache 和 mcentral，通过 mheap 分配。

```go
func mallocgc(...) {
    } else {
        var s *mspan
        shouldhelpgc = true
        systemstack(func() {
            s = largeAlloc(size, needzero)
        })
        s.freeindex = 1
        s.allocCount = 1
        x = unsafe.Pointer(s.base())
        size = s.elemsize
    }
    ...
}

func largeAlloc(size uintptr, needzero bool) *mspan {
    ...
    npages := size >> _PageShift
    if size&_PageMask != 0 {
        npages++
    }
    ...
    s := mheap_.alloc(npages, 0, true, needzero)
    if s == nil {
        throw("out of memory")
    }
    s.limit = s.base() + size
    heapBitsForSpan(s.base()).initSpan(s)
    return s
}
```

对于大于 32 K 的内存分配都是分配整数页，先右移然后低位与计算需要的页数。

#### 4.3 size 介于 16 和 32K

对于 size 介于 16 ~ 32K byte 的内存分配先计算应该分配的 sizeclass，然后去 mcache 里面 `alloc[sizeclass]` 申请，如果 `mcache.alloc[sizeclass]` 不足以申请，则 mcache 向 mcentral 申请，然后再分配。mcentral 给 mcache 分配完之后会判断自己需不需要扩充，如果需要则想 mheap 申请。

```go
func mallocgc(...) {
        ...
        } else {
            var sizeclass uint8
            
            //计算 sizeclass
            if size <= smallSizeMax-8 {
                sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
            } else {
                sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-       1)/largeSizeDiv]
            }
            size = uintptr(class_to_size[sizeclass])
            span := c.alloc[sizeclass]
            //从对应的 span 里面分配一个 object 
            v := nextFreeFast(span)
            if v == 0 {
                v, span, shouldhelpgc = c.nextFree(sizeclass)
            }
            x = unsafe.Pointer(v)
            if needzero && span.needzero != 0 {
                memclrNoHeapPointers(unsafe.Pointer(v), size)
            }
        }
}
```

我们首先看一下如何计算 sizeclass 的，预先定义了两个数组：size_to_class8 和 size_to_class128。 数组 size_to_class8，其第 i 个值表示地址区间 `( (i-1)*8, i*8 ] (smallSizeDiv = 8)` 对应的 sizeclass，size_to_class128 类似。小于 `1024 - 8 = 1016` （smallSizeMax=1024），使用 size_to_class8，否则使用数组 size_to_class128。看一下数组具体的值：0, 1, 2, 3, 3, 4, 4…。举个例子，比如要分配 17 byte 的内存 （16 byte 以下的使用 mcache.tiny 分配），`sizeclass = size_to_calss8[(17+7)/8] = size_to_class8[3] = 3`。不得不说这种用空间换时间的策略确实提高了运行效率。

计算出 sizeclass，那么就可以去 `mcache.alloc[sizeclass]` 分配了，注意这是一个 mspan 指针，真正的分配函数是 `nextFreeFast()` 函数。如下。

```go
// nextFreeFast returns the next free object if one is quickly available.
// Otherwise it returns 0.
func nextFreeFast(s *mspan) gclinkptr {
    theBit := sys.Ctz64(s.allocCache) // Is there a free object in the allocCache?
    if theBit < 64 {
        result := s.freeindex + uintptr(theBit)
        if result < s.nelems {
            freeidx := result + 1
            if freeidx%64 == 0 && freeidx != s.nelems {
                return 0
            }
            s.allocCache >>= (theBit + 1)
            s.freeindex = freeidx
            v := gclinkptr(result*s.elemsize + s.base())
            s.allocCount++
            return v
        }
    }
    return 0
}
```

allocCache 这里是用位图表示内存是否可用，1 表示可用。然后通过 span 里面的 freeindex 和 elemsize 来计算地址即可。

如果 `mcache.alloc[sizeclass]` 已经不够用了，则从 mcentral 申请内存到 mcache。

```go
// nextFree returns the next free object from the cached span if one is available.
// Otherwise it refills the cache with a span with an available object and
// returns that object along with a flag indicating that this was a heavy
// weight allocation. If it is a heavy weight allocation the caller must
// determine whether a new GC cycle needs to be started or if the GC is active
// whether this goroutine needs to assist the GC.
func (c *mcache) nextFree(sizeclass uint8) (v gclinkptr, s *mspan, shouldhelpgc bool) {
    s = c.alloc[sizeclass]
    shouldhelpgc = false
    freeIndex := s.nextFreeIndex()
    if freeIndex == s.nelems {
        // The span is full.
        if uintptr(s.allocCount) != s.nelems {
            println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
            throw("s.allocCount != s.nelems && freeIndex == s.nelems")
        }
        systemstack(func() {
            // 这个地方 mcache 向 mcentral 申请
            c.refill(int32(sizeclass))
        })
        shouldhelpgc = true
        s = c.alloc[sizeclass]
        // mcache 向 mcentral 申请完之后，再次从 mcache 申请
        freeIndex = s.nextFreeIndex()
    }

    ...
}

// nextFreeIndex returns the index of the next free object in s at
// or after s.freeindex.
// There are hardware instructions that can be used to make this
// faster if profiling warrants it.
// 这个函数和 nextFreeFast 有点冗余了
func (s *mspan) nextFreeIndex() uintptr {
    ...
}
```

mcache 向 mcentral，如果 mcentral 不够，则向 mheap 申请。

```go
func (c *mcache) refill(sizeclass int32) *mspan {
    ...
    // 向 mcentral 申请
    s = mheap_.central[sizeclass].mcentral.cacheSpan()
    ...
    return s
}

// Allocate a span to use in an MCache.
func (c *mcentral) cacheSpan() *mspan {
    ...
    // Replenish central list if empty.
    s = c.grow()
}

func (c *mcentral) grow() *mspan {
    npages := uintptr(class_to_allocnpages[c.sizeclass])
    size := uintptr(class_to_size[c.sizeclass])
    n := (npages << _PageShift) / size
    
    //这里想 mheap 申请
    s := mheap_.alloc(npages, c.sizeclass, false, true)
    ...
    return s
}
```

如果 mheap 不足，则想 OS 申请。接上面的代码 `mheap_.alloc()`

```go
func (h *mheap) alloc(npage uintptr, sizeclass int32, large bool, needzero bool) *mspan {
    ...
    var s *mspan
    systemstack(func() {
        s = h.alloc_m(npage, sizeclass, large)
    })
    ...
}

func (h *mheap) alloc_m(npage uintptr, sizeclass int32, large bool) *mspan {
    ... 
    s := h.allocSpanLocked(npage)
    ...
}

func (h *mheap) allocSpanLocked(npage uintptr) *mspan {
    ...
    s = h.allocLarge(npage)
    if s == nil {
        if !h.grow(npage) {
            return nil
        }
        s = h.allocLarge(npage)
        if s == nil {
            return nil
        }
    }
    ...
}

func (h *mheap) grow(npage uintptr) bool {
    // Ask for a big chunk, to reduce the number of mappings
    // the operating system needs to track; also amortizes
    // the overhead of an operating system mapping.
    // Allocate a multiple of 64kB.
    npage = round(npage, (64<<10)/_PageSize)
    ask := npage << _PageShift
    if ask < _HeapAllocChunk {
        ask = _HeapAllocChunk
    }

    v := h.sysAlloc(ask)
    ...
}
```

整个函数调用链如上所示，最后 sysAlloc 会调用系统调用（mmap 或者 VirtualAlloc，和初始化那部分有点类似）去向操作系统申请。

### 5. 内存回收

这里只会介绍一些简单的内存回收，更详细的垃圾回收之后会单独写一篇文章来讲。

#### 5.1 mcache 回收

mcache 回收可以分两部分：第一部分是将 alloc 中未用完的内存归还给对应的 mcentral。

```go
func freemcache(c *mcache) {
    systemstack(func() {
        c.releaseAll()
        ...

        lock(&mheap_.lock)
        purgecachedstats(c)
        mheap_.cachealloc.free(unsafe.Pointer(c))
        unlock(&mheap_.lock)
    })
}

func (c *mcache) releaseAll() {
    for i := 0; i < _NumSizeClasses; i++ {
        s := c.alloc[i]
        if s != &emptymspan {
            mheap_.central[i].mcentral.uncacheSpan(s)
            c.alloc[i] = &emptymspan
        }
    }
    // Clear tinyalloc pool.
    c.tiny = 0
    c.tinyoffset = 0
}
```

函数 releaseAll() 负责将 mcache.alloc 中各个 sizeclass 中的 mspan 归还给 mcentral。这里需要注意的是归还给 mcentral 的时候需要加锁，因为 mcentral 是全局的。除此之外将剩下的 mcache （基本是个空壳）归还给 mheap.cachealloc，其实就是把 mcache 插入 free list 表头。

```go
func (f *fixalloc) free(p unsafe.Pointer) {
    f.inuse -= f.size
    v := (*mlink)(p)
    v.next = f.list
    f.list = v
}
```

#### 5.2 mcentral 回收

当 mspan 没有 free object 的时候，将 mspan 归还给 mheap。

```go
func (c *mcentral) freeSpan(s *mspan, preserve bool, wasempty bool) bool {
    ...
    lock(&c.lock)
    ...
    if s.allocCount != 0 {
        unlock(&c.lock)
        return false
    }

    c.nonempty.remove(s)
    unlock(&c.lock)
    mheap_.freeSpan(s, 0)
    return true
}
```

#### 5.3 mheap

mheap 并不会定时向操作系统归还，但是会对 span 做一些操作，比如合并相邻的 span。

### 6. 总结

tcmalloc 是一种理论，运用到实践中还要考虑工程实现的问题。学习 Golang 源码的过程中，除了知道它是如何工作的之外，还可以学习到很多有趣的知识，比如使用变量填充 CacheLine 避免 False Sharing，利用 debruijn 序列求解 Trailing Zero（在函数中 sys.Ctz64 使用）等等。我想这就是读源码的意义所在吧。

### 7. 参考：

1.  [tcmalloc 介绍](http://legendtkl.com/2015/12/11/go-memory/)
2.  [TCMalloc : Thread-Caching Malloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)
3.  《Go 语言学习笔记》
4.  [False Sharing - wikipedia](https://en.wikipedia.org/wiki/False_sharing)



    作者：legendtkl
    链接：http://legendtkl.com/2017/04/02/golang-alloc/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
    








































