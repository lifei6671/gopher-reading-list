## 理解Go Interface

### 1 概述

Go语言中的接口很特别，而且提供了难以置信的一系列灵活性和抽象性。接口是一个自定义类型，它是一组方法的集合，要有方法为接口类型就被认为是该接口。从定义上来看，接口有两个特点:

- 接口本质是一种自定义类型，因此不要将Go语言中的接口简单理解为C++/Java中的接口，后者仅用于声明方法签名。
- 接口是一种特殊的自定义类型，其中没有数据成员，只有方法（也可以为空）。

接口是完全抽象的，因此不能将其实例化。然而，可以创建一个其类型为接口的变量，它可以被赋值为任何满足该接口类型的实际类型的值。接口的重要特性是：

- 只要某个类型实现了接口所有的方法，那么我们就说该类型实现了此接口。该类型的值可以赋给该接口的值。
- 作为1的推论，任何类型的值都可以赋值给空接口interface{}。

接口的特性是Go语言支持鸭子类型的基础，即“如果它走起来像鸭子，叫起来像鸭子（实现了接口要的方法），它就是一只鸭子（可以被赋值给接口的值）”。凭借接口机制和鸭子类型，Go语言提供了一种有利于类、继承、模板之外的更加灵活强大的选择。只要类型T的公开方法完全满足接口I的要求，就可以把类型T的对象用在需要接口I的地方。这种做法的学名叫做”Structural Typing“。


### 2 方法

Go语言中同时有函数和方法。一个方法就是一个包含了接受者的函数，接受者可以是命名类型或者结构体类型的一个值或者是一个指针。所有给定类型的方法属于该类型的方法集。

```go
type User struct {
  Name  string
  Email string
}

func (u User) Notify() error
// User 类型的值可以调用接受者是值的方法
damon := User{"AriesDevil", "ariesdevil@xxoo.com"}
damon.Notify()

// User 类型的指针同样可以调用接受者是值的方法
alimon := &User{"A-limon", "alimon@ooxx.com"}
alimon.Notify()
```

`User`的结构体类型，定义了一个该类型的方法叫做`Notify`，该方法的接受者是一个`User`类型的值。要调用`Notify`方法我们需要一个 `User`类型的值或者指针。Go调用和解引用指针使得调用可以被执行。注意，当接受者不是一个指针时，该方法操作对应接受者的值的副本(意思就是即使你使用了指针调用函数，但是函数的接受者是值类型，所以函数内部操作还是对副本的操作，而不是指针操作。

我们可以修改`Notify`方法，让它的接受者使用指针类型：

```go
func (u *User) Notify() error
```

再来一次之前的调用(注意：当接受者是指针时，即使用值类型调用那么函数内部也是对指针的操作。

总结：

- 一个结构体的方法的接收者可能是类型值或指针
- 如果接收者是值，无论调用者是类型值还是类型指针，修改都是值的副本
- 如果接收者是指针，则调用者修改的是指针指向的值本身。


### 3 接口实现

```go
type Notifier interface {
  Notify() error
}

func SendNotification(notify Notifier) error {
  return notify.Notify()
}

unc (u *User) Notify() error {
  log.Printf("User: Sending User Email To %s<%s>\n",
      u.Name,
      u.Email)
  return nil
}

func main() {
  user := User{
    Name:  "AriesDevil",
    Email: "ariesdevil@xxoo.com",
  }
  
  SendNotification(user)
}

// Output:
cannot use user (type User) as type Notifier in function argument:
User does not implement Notifier (Notify method has pointer receiver)
```

上述代码是编译不过的，见Output，编译错误关键信息`Notify method has pointer receiver`。 编译器不考虑我们的值是实现该接口的类型，接口的调用规则是建立在这些方法的接受者和接口如何被调用的基础上。下面的是语言规范里定义的规则，这些规则用来说明是否我们一个类型的值或者指针[实现](http://golang.org/ref/spec#Method_sets)了该接口：

- 类型 *T 的可调用方法集包含接受者为 *T 或 T 的所有方法集
- 类型 T 的可调用方法集包含接受者为 T 的所有方法
- 类型 T 的可调用方法集不包含接受者为 *T 的方法

也就是说：

- 接收者是指针 *T 时，接口的实例必须是指针
- 接收者是值 T 时，接口的实例可以是指针也可以是值


### 4 空接口与nil

空接口(`interface{}`)不包含任何的method，正因为如此，所有的类型都实现了`interface{}`。`interface{}`对于描述起不到任何的作用(因为它不包含任何的method），但是`interface{}`在我们需要存储任意类型的数值的时候相当有用，因为它可以存储任意类型的数值。它有点类似于C语言的`void*`类型。

Go语言中的nil在概念上和其它语言的null、None、nil、NULL一样，都指代零值或空值。nil是预先说明的标识符，也即通常意义上的关键字。nil只能赋值给指针、channel、func、interface、map或slice类型的变量。如果未遵循这个规则，则会引发panic。

在底层，interface作为两个成员来实现，一个类型(type)和一个值(data)。参考官方文档翻译Go中error类型的nil值和nil。


```go
import (
    "fmt"
    "reflect"
)
 
func main() {
    var val interface{} = int64(58)
    fmt.Println(reflect.TypeOf(val))
    val = 50
    fmt.Println(reflect.TypeOf(val))
}
```

type用于存储变量的动态类型，data用于存储变量的具体数据。在上面的例子中，第一条打印语句输出的是：int64。这是因为已经显示的将类型为int64的数据58赋值给了interface类型的变量val，所以val的底层结构应该是：(int64, 58)。我们暂且用这种二元组的方式来描述，二元组的第一个成员为type，第二个成员为data。第二条打印语句输出的是：int。这是因为字面量的整数在golang中默认的类型是int，所以这个时候val的底层结构就变成了：(int, 50)。

```go
func main() {
    var val interface{} = nil
    if val == nil {
        fmt.Println("val is nil")
    } else {
        fmt.Println("val is not nil")
    }
}
```

变量val是interface类型，它的底层结构必然是(type, data)。由于nil是untyped(无类型)，而又将nil赋值给了变量val，所以val实际上存储的是(nil, nil)。因此很容易就知道val和nil的相等比较是为true的。

进一步验证：

```go
func main() {
    var val interface{} = (*interface{})(nil)
    if val == nil {
        fmt.Println("val is nil")
    } else {
        fmt.Println("val is not nil")
    }
}
```

`(*interface{})(nil)`是将nil转成interface类型的指针，其实得到的结果仅仅是空接口类型指针并且它指向无效的地址。也就是空接口类型指针而不是空指针，这两者的区别蛮大的。

对于`(*int)(nil)`、`(*byte)(nil)`等等来说是一样的。上面的代码定义了接口指针类型变量val，它指向无效的地址(0x0)，因此val持有无效的数据。但它是有类型的`(*interface{})`。所以val的底层结构应该是：`(*interface{}, nil)`。

有时候您会看到(`*interface{}`)(nil)的应用，比如`var ptrIface = (*interface{})(nil)`，如果您接下来将ptrIface指向其它类型的指针，将通不过编译。或者您这样赋值：`*ptrIface = 123`，那样的话编译是通过了，但在运行时还是会panic的，这是因为ptrIface指向的是无效的内存地址。其实声明类似ptrIface这样的变量，是因为使用者只是关心指针的类型，而忽略它存储的值是什么。

**小结:** 无论该指针的值是什么：`(*interface{}, nil)`，这样的接口值总是非nil的，即使在该指针的内部为nil。


### 5 接口变量存储的类型

接口的变量里面可以存储任意类型的数值(该类型实现了某interface)。那么我们怎么反向知道这个变量里面实际保存了的是哪个类型的对象呢？目前常用的有两种方法：

- comma-ok断言

  value, ok = element.(T)，这里value就是变量的值，ok是一个bool类型，element是interface变量，T是断言的类型。如果element里面确实存储了T类型的数值，那么ok返回true，否则返回false。

- switch测试
  
  ```go
  switch value := element.(type) {
      case int:
          fmt.Printf("list[%d] is an int and its value is %d\n", index, value)
      case string:
           fmt.Printf("list[%d] is a string and its value is %s\n", index, value)
      ...
  ```

  element.(type)语法不能在switch外的任何逻辑里面使用，如果你要在switch外面判断一个类型就使用comma-ok。


### 6 接口与反射

反射是程序运行时检查其所拥有的结构，尤其是类型的一种能力。Go语言也提供对反射的支持。

在前面的`interface{}`与nil的底层实现已提到，在reflect包中有两个类型需要了解：Type和Value。这两个类型使得可以访问接口变量的内容，还有两个简单的函数，reflect.TypeOf和reflect.ValueOf，从接口值中分别获取reflect.Type 和reflect.Value。

如同物理中的反射，在Go语言中的反射也存在它自己的镜像。从reflect.Value可以使用Interface方法还原接口值:

```go
var x float64 = 3.4
v := reflect.ValueOf(x)

// Interface 以 interface{} 返回 v 的值。
// func (v Value) Interface() interface{}

// y 将为类型 float64
y := v.Interface().(float64) 
fmt.Println(y)
```

**声明：** 本文是收集网上一些关于Go语言中接口(interface)的说明，是一篇学习笔记，文中多处引用，参考文章列表在最后，可直接访问了解详情。



**原文：**[http://lanlingzi.cn/post/technical/2016/0803_go_interface/](http://lanlingzi.cn/post/technical/2016/0803_go_interface/)













