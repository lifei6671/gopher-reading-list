## Goroutine Local Storage

### 1 背景

最近在设计调用链与日志跟踪的API，发现相比于Java与C++，Go语言中没有原生的线程（协程）上下文，也不支持TLS（Thread Local Storage），更没有暴露API获取Goroutine的Id（后面简称GoId）。这导致无法像Java一样，把一些信息放在TLS上，用于来简化上层应用的API使用：不需要在调用栈的函数中通过传递参数来传递调用链与日志跟踪的一些上下文信息。

在Java与C++中，TLS是一种机制，指存储在线程环境内的一个结构，用来存放该线程内独享的数据。进程内的线程不能访问不属于自己的TLS，这就保证了TLS内的数据在线程内是全局共享的，而对于线程外却是不可见的。

在Java中，JDK库提供`Thread.CurrentThread()`来获取当前线程对象，提供ThreadLocal来存储与获取线程局部变量。由于Java能通过`Thread.CurrentThread()`获取当前线程，其实现的思路就很简单了，在ThreadLocal类中有一个Map，用于存储每一个线程的变量。

ThreadLocal的API提供了如下的4个方法：

```
public T get()
protected  T initialValue()
public void remove()
public void set(T value) 
```

- T get():返回此线程局部变量的当前线程副本中的值，如果这是线程第一次调用该方法，则创建并初始化此副本。
- protected T initialValue(): 返回此线程局部变量的当前线程的初始值。最多在每次访问线程来获得每个线程局部变量时调用此方法一次，即线程第一次使用get()方法访问变量的时候。如果线程先于get方法调用set(T)方法，则不会在线程中再调用initialValue方法。
- void remove(): 移除此线程局部变量的值。这可能有助于减少线程局部变量的存储需求。如果再次访问此线程局部变量，那么在默认情况下它将拥有其 initialValue。
- void set(T value)将此线程局部变量的当前线程副本中的值设置为指定值。许多应用程序不需要这项功能，它们只依赖于initialValue()方法来设置线程局部变量的值。

在Go语言中，而Google提供的解决方法是采用`golang.org/x/net/context`包来传递GoRoutine的上下文。对Go的`Context`的深入了解可参考我之前的分析：理解Go Context机制。`Context`也是能存储Goroutine一些数据达到共享，但它提供的接口是`WithValue`函数来创建一个新的`Context`对象。

```go
func WithValue(parent Context, key interface{}, val interface{}) Context {
	return &valueCtx{parent, key, val}
}

type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

从上面代码中可以看出，`Context`设置一次Value，就会产生一个`Context`对象，获取Value是先找当前`Context`存储的值，若没有再向父一级查找。获取Value可以说是多Goroutine访问安全，因为它的接口设计上，是只一个Goroutine一次设置Key/Value，其它多Goroutine只能读取Key的Value。

### 2 为什么无获取GoId接口

    This, among other reasons, to prevent programmers for simulating thread local storage using the goroutine id as a key.

官方说，就为了避免采用Goroutine Id当成`Thread Local Storage`的`Key`。

    Please don’t use goroutine local storage. It’s highly discouraged. In fact, IIRC, we used to expose Goid, but it is hidden since we don’t want people to do this.

用户经常使用GoId来实现`goroutine local storage`，而Go语言不希望用户使用`goroutine local storage`。


    when goroutine goes away, its goroutine local storage won’t be GCed. (you can get goid for the current goroutine, but you can’t get a list of all running goroutines)

不建议使用`goroutine local storage`的原因是由于不容易GC，虽然能获当前的GoId，但不能获取其它正在运行的Goroutine。

    what if handler spawns goroutine itself? the new goroutine suddenly loses access to your goroutine local storage. You can guarantee that your own code won’t spawn other goroutines, but in general you can’t make sure the standard library or any 3rd party code won’t do that.

另一个重要的原因是由于产生一个Goroutine非常地容易（而线程通用会采用线程池），新产生的Goroutine会失去访问`goroutine local storage`。需要上层应用保证不会产生新的Goroutine，但我们很难确保标准库或第三库不会这样做。


    thread local storage is invented to help reuse bad/legacy code that assumes global state, Go doesn’t have legacy code like that, and you really should design your code so that state is passed explicitly and not as global (e.g. resort to goroutine local storage)

TLS的应用是帮助重用现有那些不好（遗留）的采用全局状态的代码。而Go语言建议是重新设计代码，采用显示地传递状态而不是采用全局状态（例如采用`goroutine local storage`）。


### 3 其它手段获取GoId

虽然Go语言有意识地隐藏GoId，但目前还是有手段来获取GoId：


- 修改源代码暴露GoId，但Go语言可能随时修改源码，导致不兼容

  在标准库的runtime/proc.go（Go 1.6.3）中的newextram函数，会产生个GoId：

  ```go
  mp.lockedg = gp
  gp.lockedm = mp
  gp.goid = int64(atomic.Xadd64(&sched.goidgen, 1))
  
  ```
  
- 通过runtime.Stack来分析Stack输出信息获取GoId。

  在标准库的runtime/mprof.go（Go 1.6.3）中，runtime.Stack会获取gp对象(包含GoId)并输出整个Stack信息：
  
  ```go
  func Stack(buf []byte, all bool) int {
      if all {
          stopTheWorld("stack trace")
      }
  
      n := 0
      if len(buf) > 0 {
          gp := getg()
          sp := getcallersp(unsafe.Pointer(&buf))
          pc := getcallerpc(unsafe.Pointer(&buf))
          systemstack(func() {
              g0 := getg()
              g0.m.traceback = 1
              g0.writebuf = buf[0:0:len(buf)]
              goroutineheader(gp)
              traceback(pc, sp, 0, gp)
              if all {
                  tracebackothers(gp)
              }
              g0.m.traceback = 0
              n = len(g0.writebuf)
              g0.writebuf = nil
          })
      }
  
      if all {
          startTheWorld()
      }
      return n
  }
  ```
  从文件名就可以看出，`runtime/mprof.go`是用于做Profile分析，获取Stack肯定性能不会太好。从上面的代码来看，若第二个参数指定为true，还会STW，业务系统无论如何都无法接受。若Go语言修改了Stack的输出，分析Stack信息也会导致无法正常获取GoId。
  
- 通用runtime.Callers来给调用Stack来打标签

  代码参考：[https://github.com/jtolds/gls/blob/master/stack_tags_main.go#L43](https://github.com/jtolds/gls/blob/master/stack_tags_main.go#L43)
  
- 通过内联c或者内联汇编
  
  go版本1.5，x86_64arc下汇编，估计也不通用
  
  ```
  // func GoID() int64
  TEXT s3lib GoID(SB),NOSPLIT,$0-8
  MOVQ TLS, CX
  MOVQ 0(CX)(TLS*1), AX
  MOVQ AX, ret+0(FP)
  RET
  ```
### 4 开源goroutine local storage实现

只要有机制获取GoId，就可以像Java一样来采用全局的map实现goroutine local storage，在Github上搜索一下，发现有两个：

- [tylerb/gls](https://github.com/tylerb/gls/)

  GoId是通过runtime.Stack来分析Stack输出信息获取GoId。

- [jtolds/gls](https://github.com/jtolds/gls)

  GoId是通用runtime.Callers来给调用Stack来打标签
  
第二个有人在2013年测试过性能，数据如下：

    BenchmarkGetValue 500000 2953 ns/op
    BenchmarkSetValues 500000 4050 ns/op

上面的测试结果看似还不错，但`goroutine local storage`实现无外乎是`map+RWMutex`，存在性能瓶颈：

- Goroutine不像Thread，它的个数可以上十万并发，当这么多的Goroutine同时竞争同一把锁时，性能会急剧恶化。
- GoId是通过分析调用Stack的信息来获取，也是一个高成本的调用，一个字：慢。

不管怎么样，没有官方的GLS，的确不是很方便，第三方实现又存在性能与不兼容风险。连[tylerb/gls](https://github.com/tylerb/gls/)作者也贴出其它人的评价：


    “Wow, that’s horrifying.”
    
    “This is the most terrible thing I have seen in a very long time.”
    
    “Where is it getting a context from? Is this serializing all the requests? What the heck is the client being bound to? What are these tags? Why does he need callers? Oh god no. No no no.”

### 5 小结

Go语言官方认为TLS来存储全局状态是不好的设计，而是要显示地传递状态。Google给的解决方法是`golang.org/x/net/context`。


**原文：[http://lanlingzi.cn/post/technical/2016/0813_go_gls/](http://lanlingzi.cn/post/technical/2016/0813_go_gls/)**


