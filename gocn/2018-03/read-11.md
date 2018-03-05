## 论golang Timer Reset方法使用的正确姿势

2016年，Go语言在Tiobe编程语言排行榜上位次的大幅蹿升(2016年12月份Tiobe榜单：go位列第16位，Rating值：1.939%)。与此同时，我们也能切身感受到Go语言在世界范围蓬勃发展，其在中国地界儿上的发展更是尤为猛烈^0^：For gopher们的job变多了、网上关于Go的资料也大有“汗牛充栋”之势。作为职业Gopher^0^，要为这个生态添砖加瓦，就要多思考、多总结，关键还要做到“遇到了问题，就要说出来，给出你的见解”。每篇文章都有自己的切入角度和关注重点，因此Gopher们也无需过于担忧资料的“重复”。

这次，我来说说在使用Go标准库中Timer的Reset方法时遇到的问题。

### 一、关于Timer原理的一些说明

在网络编程方面，从用户视角看，golang表象上是一种“阻塞式”网络编程范式，而支撑这种“阻塞式”范式的则是内置于go编译后的executable file中的runtime。runtime利用网络IO多路复用机制实现多个进行网络通信的goroutine的合理调度。goroutine中的执行函数则相当于你在传统C编程中传给epoll机制的回调函数。golang一定层度上消除了在这方面“回调”这种“逆向思维”给你带来的心智负担，简化了网络编程的复杂性。

但长时间“阻塞”显然不能满足大多数业务情景，因此还需要一定的超时机制。比如：在socket层面，我们通过显式设置net.Dialer的Timeout或使用SetReadDeadline、SetWriteDeadline以及SetDeadline；在应用层协议，比如http，client通过设置timeout参数，server通过TimeoutHandler来限制操作的time limit。这些timeout机制，有些是通过runtime的网络多路复用的timeout机制实现，有些则是通过Timer实现的。

标准库中的Timer让用户可以定义自己的超时逻辑，尤其是在应对select处理多个channel的超时、单channel读写的超时等情形时尤为方便。

#### 1、Timer的创建

Timer是一次性的时间触发事件，这点与Ticker不同，后者则是按一定时间间隔持续触发时间事件。Timer常见的使用场景如下：


场景1：

```go
t := time.AfterFunc(d, f)
```

场景2:

```go
select {
    case m := <-c:
       handle(m)
    case <-time.After(5 * time.Minute):
       fmt.Println("timed out")
}

```

或：

```go
t := time.NewTimer(5 * time.Minute)
select {
    case m := <-c:
       handle(m)
    case <-t.C:
       fmt.Println("timed out")
}
```

从这两个场景中，我们可以看到Timer三种创建姿势：

```go
t:= time.NewTimer(d)
t:= time.AfterFunc(d, f)
c:= time.After(d)
```

虽然姿势不同，但背后的原理则是相通的。

Timer有三个要素：

- 定时时间：也就是那个d
- 触发动作：也就是那个f
- 时间channel： 也就是t.C

对于AfterFunc这种创建方式而言，Timer就是在超时(timer expire)后，执行函数f，此种情况下：时间channel无用。

```go
//$GOROOT/src/time/sleep.go

func AfterFunc(d Duration, f func()) *Timer {
    t := &Timer{
        r: runtimeTimer{
            when: when(d),
            f:    goFunc,
            arg:  f,
        },
    }
    startTimer(&t.r)
    return t
}

func goFunc(arg interface{}, seq uintptr) {
    go arg.(func())()
}
```

注意：从AfterFunc源码可以看到，外面传入的f参数并非直接赋值给了内部的f，而是作为wrapper function：goFunc的arg传入的。而goFunc则是启动了一个新的goroutine来执行那个外部传入的f。这是因为timer expire对应的事件处理函数的执行是在go runtime内唯一的timer events maintenance goroutine: timerproc中。为了不block timerproc的执行，必须启动一个新的goroutine。

```go
//$GOROOT/src/runtime/time.go
func timerproc() {
    timers.gp = getg()
    for {
        lock(&timers.lock)
        ... ...
            f := t.f
            arg := t.arg
            seq := t.seq
            unlock(&timers.lock)
            if raceenabled {
                raceacquire(unsafe.Pointer(t))
            }
            f(arg, seq)
            lock(&timers.lock)
        }
        ... ...
        unlock(&timers.lock)
   }
}

```

而对于NewTimer和After这两种创建方法，则是Timer在超时(timer expire)后，执行一个标准库中内置的函数：sendTime。sendTime将当前当前事件send到timer的时间Channel中，那么说这个动作不会阻塞到timerproc的执行么？答案肯定是不会的，其原因就在下面代码中：

```go
//$GOROOT/src/time/sleep.go
func NewTimer(d Duration) *Timer {
    c := make(chan Time, 1)
    t := &Timer{
        C: c,
        ... ...
    }
    ... ...
    return t
}

func sendTime(c interface{}, seq uintptr) {
    // Non-blocking send of time on c.
    // Used in NewTimer, it cannot block anyway (buffer).
    // Used in NewTicker, dropping sends on the floor is
    // the desired behavior when the reader gets behind,
    // because the sends are periodic.
    select {
    case c.(chan Time) <- Now():
    default:
    }
}

```


我们看到NewTimer中创建了一个buffered channel，size = 1。正常情况下，当timer expire，t.C无论是否有goroutine在read，sendTime都可以non-block的将当前时间发送到C中；同时，我们看到sendTime还加了双保险：通过一个select判断c buffer是否已满，一旦满了，直接退出，依然不会block，这种情况在reuse active timer时可能会遇到。

#### 2、Timer的资源释放

很多Go初学者在使用Timer时都会担忧Timer的创建会占用系统资源，比如：

有人会认为：创建一个Timer后，runtime会创建一个单独的Goroutine去计时并在expire后发送当前时间到channel里。
还有人认为：创建一个timer后，runtime会申请一个os级别的定时器资源去完成计时工作。

实际情况并不是这样。恰好近期gopheracademy blog发布了一篇 [《How Do They Do It: Timers in Go》](https://blog.gopheracademy.com/advent-2016/go-timers)，通过对timer源码的分析，讲述了timer的原理，大家可以看看。

go runtime实际上仅仅是启动了一个单独的goroutine，运行timerproc函数，维护了一个”最小堆”，定期wake up后，读取堆顶的timer，执行timer对应的f函数，并移除该timer element。创建一个Timer实则就是在这个最小堆中添加一个element，Stop一个timer，则是从堆中删除对应的element。

同时，从上面的两个Timer常见的使用场景中代码来看，我们并没有显式的去释放什么。从上一节我们可以看到，Timer在创建后可能占用的资源还包括：

- 0或一个Channel
- 0或一个Goroutine

这些资源都会在timer使用后被GC回收。

综上，作为Timer的使用者，我们要做的就是尽量减少在使用Timer时对最小堆管理goroutine和GC的压力即可，即：及时调用timer的Stop方法从最小堆删除timer element(如果timer 没有expire)以及reuse active timer。

BTW，这里还有一篇讨论[go Timer精度](https://ggaaooppeenngg.github.io/zh-CN/2016/02/09/timer%E5%9C%A8go%E5%8F%AF%E4%BB%A5%E6%9C%89%E5%A4%9A%E7%B2%BE%E7%A1%AE/)的文章，大家可以拜读一下。

### 二、Reset到底存在什么问题？

铺垫了这么多，主要还是为了说明Reset的使用问题。什么问题呢？我们来看下面的例子。这些例子主要是为了说明Reset问题，现实中很可能大家都不这么写代码逻辑。当前环境：go version go1.7 darwin/amd64。

#### 1、example1

我们的第一个example如下：

```go
//example1.go

func main() {
    c := make(chan bool)

    go func() {
        for i := 0; i < 5; i++ {
            time.Sleep(time.Second * 1)
            c <- false
        }

        time.Sleep(time.Second * 1)
        c <- true
    }()

    go func() {
        for {
            // try to read from channel, block at most 5s.
            // if timeout, print time event and go on loop.
            // if read a message which is not the type we want(we want true, not false),
            // retry to read.
            timer := time.NewTimer(time.Second * 5)
            defer timer.Stop()
            select {
            case b := <-c:
                if b == false {
                    fmt.Println(time.Now(), ":recv false. continue")
                    continue
                }
                //we want true, not false
                fmt.Println(time.Now(), ":recv true. return")
                return
            case <-timer.C:
                fmt.Println(time.Now(), ":timer expired")
                continue
            }
        }
    }()

    //to avoid that all goroutine blocks.
    var s string
    fmt.Scanln(&s)
}

```

example1.go的逻辑大致就是 一个consumer goroutine试图从一个channel里读出true，如果读出false或timer expire，那么继续try to read from the channel。这里我们每次循环都创建一个timer，并在go routine结束后Stop该timer。另外一个producer goroutine则负责生产消息，并发送到channel中。consumer中实际发生的行为取决于producer goroutine的发送行为。

example1.go执行的结果如下：

```
$go run example1.go
2016-12-21 14:52:18.657711862 +0800 CST :recv false. continue
2016-12-21 14:52:19.659328152 +0800 CST :recv false. continue
2016-12-21 14:52:20.661031612 +0800 CST :recv false. continue
2016-12-21 14:52:21.662696502 +0800 CST :recv false. continue
2016-12-21 14:52:22.663531677 +0800 CST :recv false. continue
2016-12-21 14:52:23.665210387 +0800 CST :recv true. return
```


输出如预期。但在这个过程中，我们新创建了6个Timer。

#### 2、example2

如果我们不想重复创建这么多Timer实例，而是reuse现有的Timer实例，那么我们就要用到Timer的Reset方法，见下面example2.go，考虑篇幅，这里仅列出consumer routine代码，其他保持不变：

```go
//example2.go
.... ...
// consumer routine
    go func() {
        // try to read from channel, block at most 5s.
        // if timeout, print time event and go on loop.
        // if read a message which is not the type we want(we want true, not false),
        // retry to read.
        timer := time.NewTimer(time.Second * 5)
        for {
            // timer is active , not fired, stop always returns true, no problems occurs.
            if !timer.Stop() {
                <-timer.C
            }
            timer.Reset(time.Second * 5)
            select {
            case b := <-c:
                if b == false {
                    fmt.Println(time.Now(), ":recv false. continue")
                    continue
                }
                //we want true, not false
                fmt.Println(time.Now(), ":recv true. return")
                return
            case <-timer.C:
                fmt.Println(time.Now(), ":timer expired")
                continue
            }
        }
    }()
... ...
```

按照go 1.7 doc中关于Reset使用的建议：

```go
To reuse an active timer, always call its Stop method first and—if it had expired—drain the value from its channel. For example:

if !t.Stop() {
        <-t.C
}
t.Reset(d)
```

我们改造了example1，形成example2的代码。由于producer行为并未变更，实际example2执行时，每次循环Timer在被Reset之前都没有expire，也没有fire a time to channel，因此timer.Stop的调用均返回true，即成功将timer从“最小堆”中移除。example2的执行结果如下：

```
$go run example2.go
2016-12-21 15:10:54.257733597 +0800 CST :recv false. continue
2016-12-21 15:10:55.259349877 +0800 CST :recv false. continue
2016-12-21 15:10:56.261039127 +0800 CST :recv false. continue
2016-12-21 15:10:57.262770422 +0800 CST :recv false. continue
2016-12-21 15:10:58.264534647 +0800 CST :recv false. continue
2016-12-21 15:10:59.265680422 +0800 CST :recv true. return
```

和example1并无二致。

#### 3、example3

现在producer routine的发送行为发生了变更：从以前每隔1s发送一次数据变成了每隔7s发送一次数据，而consumer routine不变：


```go
//example3.go

//producer routine
    go func() {
        for i := 0; i < 10; i++ {
            time.Sleep(time.Second * 7)
            c <- false
        }

        time.Sleep(time.Second * 7)
        c <- true
    }()

```


我们来看看example3.go的执行结果：

```
$go run example3.go
2016-12-21 15:14:32.764410922 +0800 CST :timer expired
```

程序hang住了。你能猜到在哪里hang住的吗？对，就是在drain t.C的时候hang住了：

```go
// timer may be not active and may not fired

 if !timer.Stop() {

     <-timer.C //drain from the channel

 }
 timer.Reset(time.Second * 5)
```

producer的发送行为发生了变化，Comsumer routine在收到第一个数据前有了一次time expire的事件，for loop回到loop的开始端。这时timer.Stop函数返回的不再是true，而是false，因为timer已经expire，最小堆中已经不包含该timer了，Stop在最小堆中找不到该timer，返回false。于是example3代码尝试抽干(drain)timer.C中的数据。但timer.C中此时并没有数据，于是routine block在channel recv上了。

在Go 1.8以前版本中，很多人遇到了类似的问题，并提出issue，比如：

```
time: Timer.Reset is not possible to use correctly #14038
```

不过go team认为这还是文档中对Reset的使用描述不够充分导致的，于是在Go 1.8中对Reset方法的文档做了补充，Go 1.8 beta2中Reset方法的文档改为了：

``` 
Resetting a timer must take care not to race with the send into t.C that happens when the current timer expires. If a program has already received a value from t.C, the timer is known to have expired, and t.Reset can be used directly. If a program has not yet received a value from t.C, however, the timer must be stopped and—if Stop reports that the timer expired before being stopped—the channel explicitly drained:

if !t.Stop() {
        <-t.C
}
t.Reset(d)
```

大致意思是：如果明确time已经expired，并且t.C已经被取空，那么可以直接使用Reset；如果程序之前没有从t.C中读取过值，这时需要首先调用Stop()，如果返回true，说明timer还没有expire，stop成功删除timer，可直接reset；如果返回false，说明stop前已经expire，需要显式drain channel。

#### 4、example4

我们的example3就是“time已经expired，并且t.C已经被取空，那么可以直接使用Reset ”这第一种情况，我们应该直接reset，而不用显式drain channel。如何将这两种情形合二为一，很直接的想法就是增加一个开关变量isChannelDrained，标识timer.C是否已经被取空，如果取空，则直接调用Reset。如果没有，则drain Channel。

增加一个变量总是麻烦的，RussCox也给出一个未经详尽验证的方法，我们来看看用这种方法改造的example4.go：

```go
//example4.go

//consumer
    go func() {
        // try to read from channel, block at most 5s.
        // if timeout, print time event and go on loop.
        // if read a message which is not the type we want(we want true, not false),
        // retry to read.
        timer := time.NewTimer(time.Second * 5)
        for {
            // timer may be not active, and fired
            if !timer.Stop() {
                select {
                case <-timer.C: //try to drain from the channel
                default:
                }
            }
            timer.Reset(time.Second * 5)
            select {
            case b := <-c:
                if b == false {
                    fmt.Println(time.Now(), ":recv false. continue")
                    continue
                }
                //we want true, not false
                fmt.Println(time.Now(), ":recv true. return")
                return
            case <-timer.C:
                fmt.Println(time.Now(), ":timer expired")
                continue
            }
        }
    }()
```

执行结果：

```
$go run example4.go
2016-12-21 15:38:16.704647957 +0800 CST :timer expired
2016-12-21 15:38:18.703107177 +0800 CST :recv false. continue
2016-12-21 15:38:23.706665507 +0800 CST :timer expired
2016-12-21 15:38:25.705314522 +0800 CST :recv false. continue
2016-12-21 15:38:30.70900638 +0800 CST :timer expired
2016-12-21 15:38:32.707482917 +0800 CST :recv false. continue
2016-12-21 15:38:37.711260142 +0800 CST :timer expired
2016-12-21 15:38:39.709668705 +0800 CST :recv false. continue
2016-12-21 15:38:44.71337522 +0800 CST :timer expired
2016-12-21 15:38:46.710880007 +0800 CST :recv false. continue
2016-12-21 15:38:51.713813305 +0800 CST :timer expired
2016-12-21 15:38:53.713063822 +0800 CST :recv true. return
```

我们利用一个select来包裹channel drain，这样无论channel中是否有数据，drain都不会阻塞住。看似问题解决了。

#### 5、竞争条件

如果你看过timerproc的代码，你会发现其中的这样一段代码：

```go
// go1.7
// $GOROOT/src/runtime/time.go
            f := t.f
            arg := t.arg
            seq := t.seq
            unlock(&timers.lock)
            if raceenabled {
                raceacquire(unsafe.Pointer(t))
            }
            f(arg, seq)
            lock(&timers.lock)
```


我们看到在timerproc执行f(arg, seq)这个函数前，timerproc unlock了timers.lock，也就是说f的执行并没有在锁内。

前面说过，f的执行是什么？

对于AfterFunc来说，就是启动一个goroutine，并在这个新goroutine中执行用户传入的函数；
对于After和NewTimer这种创建姿势创建的timer而言，f的执行就是sendTime的执行，也就是向t.C中send 当前时间。

注意：这时候timer expire过程中sendTime的执行与“drain channel”是分别在两个goroutine中执行的，谁先谁后，完全依靠runtime调度。于是example4.go中的看似没有问题的代码，也可能存在问题（当然需要时间粒度足够小，比如ms级的Timer）。

如果sendTime的执行发生在drain channel执行前，那么就是example4.go中的执行结果：Stop返回false（因为timer已经expire了），显式drain channel会将数据读出，后续Reset后，timer正常执行；
如果sendTime的执行发生在drain channel执行后，那么问题就来了，虽然Stop返回false（因为timer已经expire），但drain channel并没有读出任何数据。之后，sendTime将数据发到channel中。timer Reset后的Timer中的Channel实际上已经有了数据，于是当进入下面的select执行体时，”case <-timer.C:”瞬间返回，触发了timer事件，没有启动超时等待的作用。

这也是issue：*time: Timer.C can still trigger even after Timer.Reset is called #11513中问到的问题。

go官方文档中对此也有描述：

    Note that it is not possible to use Reset's return value correctly, as there is a race condition between draining the channel and the new timer expiring. Reset should always be invoked on stopped or expired channels, as described above. The return value exists to preserve compatibility with existing programs.

### 三、真的有Reset方法的正确使用姿势吗？

综合上述例子和分析，Reset的使用似乎没有理想的方案，但一般来说，在特定业务逻辑下，Reset还是可以正常工作的，就如example4那样。即便出现问题，如果了解了Reset背后的原理，问题解决起来也是会很快很准的。

文中的相关代码可以在[这里下载](https://github.com/bigwhite/experiments/tree/master/go_timer_reset)。


    作者：bigwhite
    链接：https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。