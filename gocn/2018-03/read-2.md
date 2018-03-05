## 理解Go Context机制

### 1 什么是Context

最近在公司分析gRPC源码，proto文件生成的代码，接口函数第一个参数统一是`ctx context.Context`接口，公司不少同事都不了解这样设计的出发点是什么，其实我也不了解其背后的原理。今天趁着妮妲台风妹子正面登陆深圳，全市停工、停课、停业，在家休息找了一些资料研究把玩一把。

`Context`通常被译作上下文，它是一个比较抽象的概念。在公司技术讨论时也经常会提到上下文。一般理解为程序单元的一个运行状态、现场、快照，而翻译中上下又很好地诠释了其本质，上下上下则是存在上下层的传递，上会把内容传递给下。在Go语言中，程序单元也就指的是`Goroutine`。

每个`Goroutine`在执行之前，都要先知道程序当前的执行状态，通常将这些执行状态封装在一个`Context`变量中，传递给要执行的`Goroutine`中。上下文则几乎已经成为传递与请求同生存周期变量的标准方法。在网络编程下，当接收到一个网络请求Request，处理Request时，我们可能需要开启不同的`Goroutine`来获取数据与逻辑处理，即一个请求`Request`，会在多个Goroutine中处理。而这些`Goroutine`可能需要共享Request的一些信息；同时当Request被取消或者超时的时候，所有从这个Request创建的所有Goroutine也应该被结束。

### 2 context包

Go的设计者早考虑多个Goroutine共享数据，以及多Goroutine管理机制。Context介绍请参考[Go Concurrency Patterns: Context](http://blog.golang.org/context)，[golang.org/x/net/context](http://godoc.org/golang.org/x/net/context)包就是这种机制的实现。

`context`包不仅实现了在程序单元之间共享状态变量的方法，同时能通过简单的方法，使我们在被调用程序单元的外部，通过设置ctx变量值，将过期或撤销这些信号传递给被调用的程序单元。在网络编程中，若存在A调用B的API, B再调用C的API，若A调用B取消，那也要取消B调用C，通过在A,B,C的API调用之间传递`Context`，以及判断其状态，就能解决此问题，这是为什么gRPC的接口中带上`ctx context.Context`参数的原因之一。

Go1.7(当前是RC2版本)已将原来的`golang.org/x/net/context`包挪入了标准库中，放在`$GOROOT/src/context`下面。标准库中net、net/http、os/exec都用到了context。同时为了考虑兼容，在原`golang.org/x/net/context`包下存在两个文件，go17.go是调用标准库的context包，而pre_go17.go则是之前的默认实现，其介绍请参考go程序包源码解读。

`context`包的核心就是`Context`接口，其定义如下：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
} 
```

- `Deadline`会返回一个超时时间，`Goroutine`获得了超时时间后，例如可以对某些io操作设定超时时间。

- Done方法返回一个信道（channel），当`Context`被撤销或过期时，该信道是关闭的，即它是一个表示`Context`是否已关闭的信号。

- 当Done信道关闭后，Err方法表明`Context`被撤的原因。

- `Value`可以让`Goroutine`共享一些数据，当然获得数据是协程安全的。但使用这些数据的时候要注意同步，比如返回了一个map，而这个map的读写则要加锁。

`Context`接口没有提供方法来设置其值和过期时间，也没有提供方法直接将其自身撤销。也就是说，`Context`不能改变和撤销其自身。那么该怎么通过`Context`传递改变后的状态呢？

### 3 context使用

无论是Goroutine，他们的创建和调用关系总是像层层调用进行的，就像人的辈分一样，而更靠顶部的Goroutine应有办法主动关闭其下属的Goroutine的执行（不然程序可能就失控了）。为了实现这种关系，`Context`结构也应该像一棵树，叶子节点须总是由根节点衍生出来的。

要创建Context树，第一步就是要得到根节点，`context.Background`函数的返回值就是根节点：

```go
func Background() Context
```

该函数返回空的Context，该Context一般由接收请求的第一个`Goroutine`创建，是与进入请求对应的`Context`根节点，它不能被取消、没有值、也没有过期时间。它常常作为处理`Request`的顶层`context`存在。

有了根节点，又该怎么创建其它的子节点，孙节点呢？`context`包为我们提供了多个函数来创建他们：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key interface{}, val interface{}) Context
```


函数都接收一个Context类型的参数parent，并返回一个`Context`类型的值，这样就层层创建出不同的节点。子节点是从复制父节点得到的，并且根据接收参数设定子节点的一些状态值，接着就可以将子节点传递给下层的Goroutine了。

再回到之前的问题：该怎么通过`Context`传递改变后的状态呢？使用`Context`的Goroutine无法取消某个操作，其实这也是符合常理的，因为这些Goroutine是被某个父Goroutine创建的，而理应只有父Goroutine可以取消操作。在父Goroutine中可以通过WithCancel方法获得一个cancel方法，从而获得cancel的权利。

第一个WithCancel函数，它是将父节点复制到子节点，并且还返回一个额外的`CancelFunc`函数类型变量，该函数类型的定义为：

```go
type CancelFunc func()
```

调用CancelFunc对象将撤销对应的`Context`对象，这就是主动撤销`Context`的方法。在父节点的Context所对应的环境中，通过WithCancel函数不仅可创建子节点的Context，同时也获得了该节点Context的控制权，一旦执行该函数，则该节点Context就结束了，则子节点需要类似如下代码来判断是否已结束，并退出该Goroutine：

```go 
select {
    case <-cxt.Done():
        // do some clean...
}
```

`WithDeadline`函数的作用也差不多，它返回的`Context`类型值同样是`parent`的副本，但其过期时间由`deadline`和`parent`的过期时间共同决定。当`parent`的过期时间早于传入的`deadline`时间时，返回的过期时间应与parent相同。父节点过期时，其所有的子孙节点必须同时关闭；反之，返回的父节点的过期时间则为`deadline`。

`WithTimeout`函数与`WithDeadline`类似，只不过它传入的是从现在开始`Context`剩余的生命时长。他们都同样也都返回了所创建的子Context的控制权，一个`CancelFunc`类型的函数变量。

当顶层的Request请求函数结束后，我们就可以cancel掉某个`context`，从而层层Goroutine根据判断cxt.Done()来结束。

`WithValue`函数，它返回`parent`的一个副本，调用该副本的`Value(key)`方法将得到val。这样我们不光将根节点原有的值保留了，还在子孙节点中加入了新的值，注意若存在`Key`相同，则会被覆盖。

### 3.1 小结

`context`包通过构建树型关系的`Context`，来达到上一层Goroutine能对传递给下一层Goroutine的控制。对于处理一个`Request`请求操作，需要采用`context`来层层控制`Goroutine`，以及传递一些变量来共享。

`Context`对象的生存周期一般仅为一个请求的处理周期。即针对一个请求创建一个`Context`变量（它为`Context`树结构的根）；在请求处理结束后，撤销此ctx变量，释放资源。

每次创建一个Goroutine，要么将原有的`Context`传递给Goroutine，要么创建一个子`Context`并传递给Goroutine。

Context能灵活地存储不同类型、不同数目的值，并且使多个Goroutine安全地读写其中的值。

当通过父`Context`对象创建子`Context`对象时，可同时获得子`Context`的一个撤销函数，这样父Context对象的创建环境就获得了对子`Context`将要被传递到的Goroutine的撤销权。

在子`Context`被传递到的`goroutine`中，应该对该子`Context`的Done信道（channel）进行监控，一旦该信道被关闭（即上层运行环境撤销了本`goroutine`的执行），应主动终止对当前请求信息的处理，释放资源并返回。



### 4 使用原则

Programs that use Contexts should follow these rules to keep interfaces consistent across packages and enable static analysis tools to check context propagation:
使用Context的程序包需要遵循如下的原则来满足接口的一致性以及便于静态分析。

- Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it. The Context should be the first parameter, typically named ctx；不要把Context存在一个结构体当中，显式地传入函数。Context变量需要作为第一个参数使用，一般命名为ctx；

- Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use；即使方法允许，也不要传入一个nil的Context，如果你不确定你要用什么Context的时候传一个context.TODO；

- Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions；使用context的Value相关方法只应该用于在程序和接口中传递的和请求相关的元数据，不要用它来传递一些可选的参数；

- The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines；同样的Context可以用来传递到不同的goroutine中，Context在多个goroutine中是安全的；

**原文： [http://lanlingzi.cn/post/technical/2016/0802_go_context/](http://lanlingzi.cn/post/technical/2016/0802_go_context/)**























