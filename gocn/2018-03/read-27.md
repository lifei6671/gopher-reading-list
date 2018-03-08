## go反射实践及剖析

### Go struct拷贝

在用Go做orm相关操作的时候，经常会有struct之间的拷贝。比如下面两个struct之间要拷贝共同成员B,C。这个在struct不是很大的时候从来都不是问题，直接成员拷贝即可。但是当struct的大小达到三四十个成员的时候，就要另辟蹊径了。

#### 做法一. 反射

```go
func CopyStruct(src, dst interface{}) {
	sval := reflect.ValueOf(src).Elem()
	dval := reflect.ValueOf(dst).Elem()

	for i := 0; i < sval.NumField(); i++ {
		value := sval.Field(i)
		name := sval.Type().Field(i).Name

		dvalue := dval.FieldByName(name)
		if dvalue.IsValid() == false {
			continue
		}
		dvalue.Set(value)
	}
}
```

#### 做法二. json

先encode成json，再decode，其实golang的json包内部实现也是使用的反射，所以再大型项目中可以考虑使用ffjson来作为替代方案。

```go
func main() {
	a := &A{1, "a", 1}
	//	b := &B{"b",2,2}

	aj, _ := json.Marshal(a)
	b := new(B)
	_ = json.Unmarshal(aj, b)

	fmt.Printf("%+v", b)
}
```


关于golang中的反射机制一直是大家诟病挺多的。因为反射中使用了类型的枚举，所以效率比较低，在高性能场景中应该尽量规避，但是，对于大部分应用场景来说，牺牲一点性能来极大减少代码行数，或者说提高开发效率都是值得的。

### Reflect

关于Reflect的接口可以参考golang的文档，也可以直接看go的源码。reflect的核心是两个，一个是Type，一个是Value。reflect的使用一般都是以下面语句开始。

### Value

Value的定义很简单，如下。

```go
type Value struct {
    typ  *rtype
    ptr  unsafe.Pointer     //pointer-valued data or pointer to data
    flag    //metedata 
}
```

下面针对object的类型不同来看一下reflect提供的方法。

**built-in type**

reflect针对基本类型提供了如下方法，以Float()为例展开。

```go
//读
func (v Value) Float() float64 {
	k := v.kind()
	switch k {
	case Float32:
		return float64(*(*float32)(v.ptr))
	case Float64:
		return *(*float64)(v.ptr)
	}
	panic(&ValueError{"reflect.Value.Float", v.kind()})
}
func (v Value) Bool() bool
func (v Value) Bytes() []byte
func (v Value) Int() int64
func (v Value) Uint() uint64
func (v Value) String() string
func (v Value) Complex() complex128
//写
func (v Value) Set(x Value)
func (v Value) SetBool(x bool)
func (v Value) SetBytes(x []byte)
func (v Value) SetCap(n int)
func (v Value) SetComplex(x complex128)
func (v Value) SetFloat(x float64)
func (v Value) SetInt(x int64) {
    v.mustBeAssignable()
	switch k := v.kind(); k {
	default:
		panic(&ValueError{"reflect.Value.SetInt", v.kind()})
	case Int:
		*(*int)(v.ptr) = int(x)
	case Int8:
		*(*int8)(v.ptr) = int8(x)
	case Int16:
		*(*int16)(v.ptr) = int16(x)
	case Int32:
		*(*int32)(v.ptr) = int32(x)
	case Int64:
		*(*int64)(v.ptr) = x
	}
}
func (v Value) SetLen(n int)
func (v Value) SetMapIndex(key, val Value)
func (v Value) SetPointer(x unsafe.Pointer)
func (v Value) SetString(x string)
func (v Value) SetUint(x uint64)
```

可以看到内部Float内部实现先通过kind()判断value的类型，然后通过指针取数据并做类型转换。kind()的定义如下

```go
func (f flag) kind() Kind {
    return Kind(f & flagKindMask)
}
```

flag是Value的内部成员，所以方法也被继承过来。通过实现不难猜到reflect对不同的类型是通过一个整数来实现的。我们来验证一下，在type.go文件中找到Kind定义，注意这个地方Kind()只是一个类型转换。

```go
type Kind uint

const (
    Invalid Kind = iota
    Bool
    Int
    Int8
    Int16
    Int32
    ...
)
```

紧挨着Kind定义的下面就是各种类型的表示：`Bool=1, Int=2`，…再来看一下`f & flagKindMask`的意思。再value.go文件中找到flagKindMask的定义:

```go
const (
	flagKindWidth        = 5 // there are 27 kinds
	flagKindMask    flag = 1<<flagKindWidth - 1
	...
}
```

所以这句`f & flagKindMask` 的意思就是最f的低5位，也就是对应了上面的各种类型。

上面说完了数据的读接口，其实写接口也很类似。唯一的区别在于reflect还提供了一个Set()方法，就是说我们自己去保证数据的类型正确性。

**Struct**

其实使用反射的很多场景都是struct。reflect针对struct提供的函数方法如下：

```go
func (v Value) Elem() Value     //返回指针或者interface包含的值
func (v Value) Field(i int) Value       //返回struct的第i个field
func (v Value) FieldByIndex(index []int) Value      //返回嵌套struct的成员
func (v Value) FieldByName(name string) Value       //通过成员名称返回对应的成员
func (v Value) FieldByNameFunc(match func(string) bool) Value   //只返回满足函数match的第一个field

```

通过上面的方法不出意外就可以取得对应是struct field了。

**其他类型：Array, Slice, String**

对于其他类型，reflect也提供了获得其内部成员的方法。

```go
func (v Value) Len() int    //Array, Chan, Map, Slice, or String
func (v Value) Index(i int) Value   //Array, Slice, String
func (v Value) Cap() int            //Array, Chan, Slice
func (v Value) Close()              //Chan
func (v Value) MapIndex(key Value) Value    //Map
func (v Value) MapKeys() []Value            //Map
```

**函数调用**

reflect当然也可以实现函数调用，下面是一个简单的例子。

```go
func main() {
    var f = func() {
        fmt.Println("hello world")
    }
    
    fun := reflect.ValueOf(f)
    fun.Call(nil)
}
//Output
hello world
```

当然我们还可以通过struct来调用其方法，需要注意的一点是通过反射调用的函数必须是外部可见的（首字母大写）。

```go
type S struct {
    A int
}

func (s S) Method1() {
    fmt.Println("method 1")
}

func (s S) Method2() {
    fmt.Println("method 2")
}

func main() {
    var a S
    S := reflect.ValueOf(a)

    fmt.Println(S.NumMethod())

    m1 :=S.Method(0)
    m1.Call(nil)

    m2 := S.MethodByName("Method2")
    m2.Call(nil)
}
```

总结一下reflect提供了函数调用的相关接口如下：

```go
func (v Value) Call(in []Value) []Value
func (v Value) CallSlice(in []Value) []Value
func (v Value) Method(i int) Value      //v's ith function
func (v Value) NumMethod() int
func (v Value) MethodByName(name string) Value
```

**Type**

Type是反射中另外重要的一部分。Type是一个接口，里面包含很多方法，通常我们可以通过`reflect.TypeOf(obj)`取得obj的类型对应的Type接口。接口中很多方法都是对应特定类型的，所以调用的时候需要注意。

```go
type Type interface {
	Align() int
	FieldAlign() int
	Method(int) Method
	MethodByName(string) (Method, bool)
	NumMethod() int
	...
```

另外reflect包还为我们提供了几个生成特定类型的Type接口的方法。这里就不一一列举了。

### 小结

关于reflect提供的接口需要注意的一点就是，一定要保证类型是匹配的，如果不匹配将导致panic。关于Value的主要接口都在这，本来还想写一下Type以及内部的实现机制的，只能放到下篇再写了。

#

    作者：legendtkl
    链接：http://legendtkl.com/2016/08/03/go-reflect-value/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。












