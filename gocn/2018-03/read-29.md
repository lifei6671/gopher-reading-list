## 谈一谈Go的interface和reflect

在上一篇的go泛型编程里面简单地介绍了一下interface的使用，下面继续。

### 函数传参slice

先从一个问题开始：

    有一个函数的输入参数是slice，但是slice里面的元素类型未知，如何来定义这个函数？

听上去很简单，很多人可能会写出下面的代码。

```go
func MethodTakeinSlice(in []interface{}){...}
...
slice := []int{1,2,3}
MethodTakeinSlice(slice)
```

很遗憾，这样会得到一个错误信息：`cannot use slice (type []int) as type []interface {} in argument to MethodTakeinSlice`。一个简单的解决方案是写一个convert函数。


```go
func convert(in []AnyType) (out []interface{}) {
    out = make([]interface{}, len(in))
    for i, v := range in {
        out[i] = v
    }
    return
}
```

但这样相当于泛型的优势又没了，因为每一种特定类型都得做一个转换。这里就引入了go语言的另外一个特性reflect。

### reflect

reflect，中文一般叫做反射。反射机制是指在运行时态能够调用对象的方法和属性。很多人比较熟悉的是Java的反射机制，其实go语言中也提供了反射机制，import reflect就可以使用。在go语言中，主要用在函数的参数是interface{}类型，运行时根据传入的参数的特定类型执行不同的动作。reflect针对很多数据类型都提供了一些方法以及属性。下面以Struct为例。

```go
//file: src/reflect/type.go
// A StructField describes a single field in a struct.
type StructField struct {
    // Name is the field name.
    Name string
    // PkgPath is the package path that qualifies a lower case (unexported)
    // field name.  It is empty for upper case (exported) field names.
    // See https://golang.org/ref/spec#Uniqueness_of_identifiers
    PkgPath string
    
    Type      Type      // field type
    Tag       StructTag // field tag string
    Offset    uintptr   // offset within struct, in bytes
    Index     []int     // index sequence for Type.FieldByIndex
    Anonymous bool      // is an embedded field
}
```

这是reflect为Struct统一定义的属性，我们可以通过这些属性来操作struct，下面是一个简单的例子。

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    kelu := Person{"kelu", 25}
    t := reflect.TypeOf(kelu)
    n := t.NumField()
    for i := 0; i < n; i++ {
        fmt.Println(t.Field(i).Name)
        fmt.Println(t.Field(i).Type)
    }
}
//output as follow
//name
//string
//age
//int
```

### 用reflect解决上面的问题

用reflect实现上面的问题无非就是通过reflect检验传入interface{}的类型，然后操作。

```go
package main

import (
    "fmt"
    "reflect"
)

type Person struct {
    name string
    age  int
}

func Method(in interface{}) (ok bool) {
    v := reflect.ValueOf(in)
    if v.Kind() == reflect.Slice {
    	ok = true
    }
    else {
    	//panic
    }
    
    num := v.Len()
    for i := 0; i < num; i++ {
    	fmt.Println(v.Index(i).Interface())
    }
    return ok
}

func main() {
    s := []int{1, 3, 5, 7, 9}
    b := []float64{1.2, 3.4, 5.6, 7.8}
    Method(s)
    Method(b)
}
```

其中refelct.Slice表示Slice，还有其他数据类型，一共25种。具体实现将在下面一节细讲。

### reflect源码浅析

这部分内容将结合go语言的源码简单说一下reflect的实现，是作为为那些好奇心比较强的人写的。

**reflect中数据类型表示**

```go
const (
    Invalid Kind = iota
    Bool
    Int
    Int8
    ...
    Ptr
    Slice
    String
    Struct
    UnsafePointer
)
```

其中Bool为1，Int为2，依次递增1。可以用代码验证一下。

```go
fmt.Printf("%d\n", reflect.Bool)	//output: 1
fmt.Println(reflect.Bool)	//output: bool
```

下面为上面会输出bool呢，这是因为type.go中实现了String()方法。

```go
func (k Kind) String() string {
    if int(k) < len(kindNames) {
    	return kindNames[k]
    }
    return "kind" + strconv.Itoa(int(k))
}
...
var kindNames = []string{
    Invalid:       "invalid",
    Bool:          "bool",
    Int:           "int",
    Int8:          "int8",
    Int16:         "int16",
    Int32:         "int32",
    Int64:         "int64",
    ...
    Interface:     "interface",
    Map:           "map",
    Ptr:           "ptr",
    Slice:         "slice",
    String:        "string",
    Struct:        "struct",
    UnsafePointer: "unsafe.Pointer",
}
```

### 类型反射解析

我们按ValueOf()函数的流程走一遍。

```go
func ValueOf(i interface{}) Value {
    if i == nil {
    	return Value{}
    }
    
    // TODO(rsc): Eliminate this terrible hack.
    // In the call to unpackEface, i.typ doesn't escape,
    // and i.word is an integer.  So it looks like
    // i doesn't escape.  But really it does,
    // because i.word is actually a pointer.
    escapes(i)
    
    return unpackEface(i)
}
```

escapes(i)是一个特殊处理，可以先不管，我们先看一下Value struct结构和unpackEface()函数。

```go
// unpackEface converts the empty interface i to a Value.
type Value struct {
    // typ holds the type of the value represented by a Value.
    typ *rtype
    
    // Pointer-valued data or, if flagIndir is set, pointer to data.
    // Valid when either flagIndir is set or typ.pointers() is true.
    ptr unsafe.Pointer
    
    //too much comment, delete it
    flag
}
...
func unpackEface(i interface{}) Value {
    e := (*emptyInterface)(unsafe.Pointer(&i))
    // NOTE: don't read e.word until we know whether it is really a pointer or not.
    t := e.typ
    if t == nil {
    	return Value{}
    }
    f := flag(t.Kind())
    if ifaceIndir(t) {
    	f |= flagIndir
    }
    return Value{t, unsafe.Pointer(e.word), f}
}
```

Value struct结构是核心，rtype是数据的底层实现，flag是元数据，可用用来表征Value的数据类型。unpackEface是把interface{}转换成Value数据。代码中ifaceIndir()用来确定是不是指针。
我们上面代码中还出现了Kind()，Kind()内不是实现的是kind()，从下面代码可以看出主要就是一个&的运算。这里还有个一个小细节。1<<flagKindWidth - 1在这里的结果为(1<<flagKindWidth) - 1，而C语言中的结果为1<<(flagKindWidth-1)。

```go
const (
    flagKindWidth        = 5 // there are 27 kinds
    flagKindMask    flag = 1<<flagKindWidth - 1
    flagStickyRO    flag = 1 << 5
    flagEmbedRO     flag = 1 << 6
    flagIndir       flag = 1 << 7
    flagAddr        flag = 1 << 8
    flagMethod      flag = 1 << 9
    flagMethodShift      = 10
    flagRO          flag = flagStickyRO | flagEmbedRO
)

func (f flag) kind() Kind {
    return Kind(f & flagKindMask)
}
```

### reference

- [Golang.org](https://golang.org/)：pkg reflect, fmt, unsafe, etc
- [Golang take slices of any type as input parameter](https://ahmetalpbalkan.com/blog/golang-take-slices-of-any-type-as-input-parameter/)
- [Go reflect Example](http://merbist.com/2011/06/27/golang-reflection-exampl/)
- [Go unsafe.Pointer](http://learngowith.me/gos-pointer-pointer-type/)


#

    作者：legendtkl
    链接：http://legendtkl.com/2015/11/28/go-interface-reflect/
    著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。













