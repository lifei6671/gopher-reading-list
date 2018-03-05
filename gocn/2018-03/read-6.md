## golang IO包的妙用

golang标准库对io的抽象非常精巧，各个组件可以随意组合，可以作为接口设计的典范。这篇文章结合一个实际的例子来和大家分享一下。

### 背景

以一个RPC的协议包来说，每个包有如下结构

```go
type Packet struct {
    TotalSize uint32
    Magic     [4]byte
    Payload   []byte
    Checksum  uint32
}
```

其中`TotalSize`是整个包除去`TotalSize`后的字节数， `Magic`是一个固定长度的字串，`Payload`是包的实际内容，包含业务逻辑的数据。
`Checksum`是对`Magic`和`Payload`的`adler32`校验和。

### 编码(encode)

我们使用一个原型为`func EncodePacket(w io.Writer, payload []byte) error`的函数来把数据打包，结合`encoding/binary`我们很容易写出第一版，演示需要，错误处理方面就简化处理了。

```go
var RPC_MAGIC = [4]byte{'p', 'y', 'x', 'i'}

func EncodePacket(w io.Writer, payload []byte) error {
    // len(Magic) + len(Checksum) == 8
    totalsize := uint32(len(payload) + 8)
    // write total size
    binary.Write(w, binary.BigEndian, totalsize)

    // write magic bytes
    binary.Write(w, binary.BigEndian, RPC_MAGIC)

    // write payload
    w.Write(payload)

    // calculate checksum
    var buf bytes.Buffer
    buf.Write(RPC_MAGIC[:])
    buf.Write(payload)
    checksum := adler32.Checksum(buf.Bytes())

    // write checksum
    return binary.Write(w, binary.BigEndian, checksum)
}
```

在上面的实现中，为了计算`checksum`，我们使用了一个内存`buffer`来缓存数据，最后把所有的数据一次性读出来算`checksum`，考虑到计算`checksum`是一个不断`update`地过程，我们应该有方法直接略过内存`buffer`而计算`checksum`。

查看[hash/adler32](https://link.jianshu.com/?t=http://godoc.org/hash/adler32#New)我们得知，我们可以构造一个[Hash32](https://link.jianshu.com/?t=http://godoc.org/hash#Hash32)的对象，这个对象内嵌了一个[Hash](https://link.jianshu.com/?t=http://godoc.org/hash#Hash)的接口，这个接口的定义如下：


```go
type Hash interface {
    // Write (via the embedded io.Writer interface) adds more data to the running hash.
    // It never returns an error.
    io.Writer

    // Sum appends the current hash to b and returns the resulting slice.
    // It does not change the underlying hash state.
    Sum(b []byte) []byte

    // Reset resets the Hash to its initial state.
    Reset()

    // Size returns the number of bytes Sum will return.
    Size() int

    // BlockSize returns the hash's underlying block size.
    // The Write method must be able to accept any amount
    // of data, but it may operate more efficiently if all writes
    // are a multiple of the block size.
    BlockSize() int
}
```

这是一个通用的计算hash的接口，标准库里面所有计算hash的对象都实现了这个接口，比如md5, crc32等。由于`Hash`实现了`io.Writer`接口，因此我们可以把所有要计算的数据像写入文件一样写入到这个对象中，最后调用`Sum(nil)`就可以得到最终的`hash`的`byte`数组。利用这个思路，第二版可以这样写:

```go
func EncodePacket2(w io.Writer, payload []byte) error {
    // len(Magic) + len(Checksum) == 8
    totalsize := uint32(len(RPC_MAGIC) + len(payload) + 4)
    // write total size
    binary.Write(w, binary.BigEndian, totalsize)

    // write magic bytes
    binary.Write(w, binary.BigEndian, RPC_MAGIC)

    // write payload
    w.Write(payload)

    // calculate checksum
    sum := adler32.New()
    sum.Write(RPC_MAGIC[:])
    sum.Write(payload)
    checksum := sum.Sum32()

    // write checksum
    return binary.Write(w, binary.BigEndian, checksum)
}
```

注意这次的变化，前面写入TotalSize，Magic，Payload部分没有变化，在计算checksum的时候去掉了bytes.Buffer，减少了一次内存申请和拷贝。

考虑到`sum`和w都是`io.Writer`，利用神奇的[io.MultiWriter](https://link.jianshu.com/?t=http://godoc.org/io#MultiWriter)，我们可以这样写

```go
func EncodePacket(w io.Writer, payload []byte) error {
    // len(Magic) + len(Checksum) == 8
    totalsize := uint32(len(RPC_MAGIC) + len(payload) + 4)
    // write total size
    binary.Write(w, binary.BigEndian, totalsize)

    sum := adler32.New()
    ww := io.MultiWriter(sum, w)
    // write magic bytes
    binary.Write(ww, binary.BigEndian, RPC_MAGIC)

    // write payload
    ww.Write(payload)

    // calculate checksum
    checksum := sum.Sum32()

    // write checksum
    return binary.Write(w, binary.BigEndian, checksum)
}
```

注意`MultiWriter`的使用，我们把`w`和`sum`利用`MultiWriter`绑在了一起创建了一个新的`Writer`，向这个`Writer`里面写入数据就同时向`w`和`sum`里面都写入数据，这样就完成了发送数据和计算`checksum`的同步进行，而对于`binary.Write`来说没有任何区别，因为它需要的是一个实现了`Write`方法的对象。


### 解码(decode)

基于上面的思想，解码也可以把接收数据和计算checksum一起进行，完整代码如下

```go
func DecodePacket(r io.Reader) ([]byte, error) {
    var totalsize uint32
    err := binary.Read(r, binary.BigEndian, &totalsize)
    if err != nil {
        return nil, errors.Annotate(err, "read total size")
    }

    // at least len(magic) + len(checksum)
    if totalsize < 8 {
        return nil, errors.Errorf("bad packet. header:%d", totalsize)
    }

    sum := adler32.New()
    rr := io.TeeReader(r, sum)

    var magic [4]byte
    err = binary.Read(rr, binary.BigEndian, &magic)
    if err != nil {
        return nil, errors.Annotate(err, "read magic")
    }
    if magic != RPC_MAGIC {
        return nil, errors.Errorf("bad rpc magic:%v", magic)
    }

    payload := make([]byte, totalsize-8)
    _, err = io.ReadFull(rr, payload)
    if err != nil {
        return nil, errors.Annotate(err, "read payload")
    }

    var checksum uint32
    err = binary.Read(r, binary.BigEndian, &checksum)
    if err != nil {
        return nil, errors.Annotate(err, "read checksum")
    }

    if checksum != sum.Sum32() {
        return nil, errors.Errorf("checkSum error, %d(calc) %d(remote)", sum.Sum32(), checksum)
    }
    return payload, nil
}
```

上面代码中，我们使用了[io.TeeReader](https://link.jianshu.com/?t=http://godoc.org/io#TeeReader)，这个函数的原型为`func TeeReader(r Reader, w Writer) Reader`，它返回一个`Reader`，这个`Reader`是参数r的代理，读取的数据还是来自`r`，不过同时把读取的数据写入到`w`里面。

### 一切皆文件

unix下有一切皆文件的思想，golang把这个思想贯彻到更远，因为本质上我们对文件的抽象就是一个可读可写的一个对象，也就是实现了`io.Writer`和`io.Reader`的对象我们都可以称为文件，在上面的例子中无论是`EncodePacket`还是`DecodePacket`我们都没有假定编码后的数据是发送到socket，还是从内存读取数据解码，因此我们可以这样调用EncodePacket

```go
conn, _ := net.Dial("tcp", "127.0.0.1:8000")
EncodePacket(conn, []byte("hello"))
```

把数据直接发送到socket，也可以这样

```go
conn, _ := net.Dial("tcp", "127.0.0.1:8000")
bufconn := bufio.NewWriter(conn)
EncodePacket(bufconn, []byte("hello"))
```

对socket加上一个buffer来增加吞吐量，也可以这样

```go
conn, _ := net.Dial("tcp", "127.0.0.1:8000")
zip := zlib.NewWriter(conn)
bufconn := bufio.NewWriter(conn)
EncodePacket(bufconn, []byte("hello"))
```

加上一个zip压缩，还可以利用加上[crypto/aes](https://link.jianshu.com/?t=http://godoc.org/crypto/aes)来个AES加密...

在这个时候，文件已经不再局限于io，可以是一个内存buffer，也可以是一个计算hash的对象，甚至是一个计数器，流量限速器。golang灵活的接口机制为我们提供了无限可能。

### 结尾

我一直认为一个好的语言一定有一个设计良好的标准库，golang的标准库是作者们多年系统编程的沉淀，值得我们细细品味。


作者：icexin
链接：[https://www.jianshu.com/p/8c33f7c84509](https://www.jianshu.com/p/8c33f7c84509)
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



















































