# bufio
包 `bufio` 是封装了 `io` 操作的带一个缓冲区的 `Reader` 和 `Writer`，为我们操作文本 `io` 提供了一些便捷。

## Reader & Writer
`Reader` 是封装了 `io.Reader` 接口的一个对象，其本身也实现了 `io.Reader` 接口。内部实现是一个环形队列，每次读取的时候会将数据读取到环形队列的缓冲区（对于读取字节数小于缓冲区大小，并且缓冲区没有数据的时候，会直接将数据读取到用户传递的切片中）。 `Writer` 的思想与 `Reader` 是相同的。

```
type Reader struct {
    buf          []byte
    rd           io.Reader // 一个Reader接口，可以是文件句柄，socket等
    r, w         int       // 缓冲区的读写位置
    err          error
    lastByte     int // last byte read for UnreadByte; -1 means invalid
    lastRuneSize int // size of last rune read for UnreadRune; -1 means invalid
}
```
初始化的方法有以下两种：
```
// 默认创建4096字节的缓冲区（此函数调用下面的函数，参数默认为4096）
func NewReader(rd io.Reader) *Reader
// 根据第二个参数创建缓冲区
func NewReaderSize(rd io.Reader, size int) *Reader {
    // 判断传进来的是不是已经是我们的带缓冲区的Reader了
    // 因为我们封装的是接口，所以这里缓冲区够用的话，就不让套娃了
    b, ok := rd.(*Reader)
    if ok && len(b.buf) >= size {
        return b
    }
    if size < minReadBufferSize {
        size = minReadBufferSize
    }
    r := new(Reader)
    // 这里创建了slice
    r.reset(make([]byte, size), rd)
    return r
}
```
这个工具包常用来对 `socket` 进行一层封装。将 `net.Conn` 作为参数生成 `Writer` 和 `Reader` 对象。使用相应的方法可以省去我们直接读取 `socket` 需要自己去管理缓冲区的麻烦。
```
// nsq中的用法
......
// 这个readLen 内部其实也是使用io.ReadFull。我们传进去一个长度为我们协议的header。
bodyLen, err := readLen(client.Reader, client.lenSlice)
......
// 根据header数据拿到bodyLen，读取所有数据到body
body := make([]byte, bodyLen)
    _, err = io.ReadFull(client.Reader, body)
```
可以看到，以上用法非常轻松的把我们从读取 `socket` 中解放出来，只需要关心自己定的协议即可。
```
// nsq中的用法
// 先写入协议的头
binary.BigEndian.PutUint32(beBuf, size)
n, err := w.Write(beBuf)
if err != nil {
    return n, err
}

binary.BigEndian.PutUint32(beBuf, uint32(frameType))
n, err = w.Write(beBuf)
if err != nil {
    return n + 4, err
}
// 最后写入body
n, err = w.Write(data)
```
使用 `Writer` 要注意，调用了 `Write` 函数不一定真的写入了 `socket`，调用 `Flush` 可以确保数据真的被写入了。 `nsq` 使用一个定时器来定时调用此函数进行刷新。注：`nsq` 写入可能有多个协程写，因此有一个写的锁，保证数据不会被混淆。  
上面的用法是经典的为 `socket` 添加一个缓冲区，但是自己的数据还是需要重新分配内存（例如读取 `body` 时，内存拷贝也就多了一次，而且一个socket分配两块内存，socket关闭，这个内存需要gc来回收）。而毛大的 `goim` 自己实现了内存池，并且给 `bufio` 这两个读写对象增加了新的接口，复用内存。
```
// 从外面传进来一个切片  不要让Reader对象再自动生成
func (b *Reader) ResetBuffer(r io.Reader, buf []byte) {
    b.reset(buf, r)
}

// 返回值与环形队列共享一个底层数组，这样做也有问题，就是如果你数据还没使用，可能被下一次读的数据覆盖
func (b *Reader) Pop(n int) ([]byte, error) {
    d, err := b.Peek(n)
    if err == nil {
        b.r += n
        return d, err
    }
    return nil, err
}

// 与读的Pop一样，调用这个函数将写的环形队列返回。缺点也是相同的。
func (b *Writer) Peek(n int) ([]byte, error) {
    if n < 0 {
        return nil, ErrNegativeCount
    }
    if n > len(b.buf) {
        return nil, ErrBufferFull
    }
    for b.Available() < n && b.err == nil {
        b.flush()
    }
    if b.err != nil {
        return nil, b.err
    }
    d := b.buf[b.n : b.n+n]
    b.n += n
    return d, nil
}
```
上面的这些缺点是可以从调用角度解决的。比如 `goim` 是这样做的，同一个 `socket` 读协程只有一个，按照顺序读，每次读完就立刻处理数据，处理完毕再进行下一次读。对于写是相同的道理。这样相比于 `nsq` 的用法就少了一次拷贝。最重要的是可以自己管理环形队列这块内存，`goim` 就是自己写了个内存池来管理这些内存，避免gc频繁。

`goim` 这一块的代码非常厉害，不仅拷贝次数减少，内存使用减少，gc次数也会减少。
