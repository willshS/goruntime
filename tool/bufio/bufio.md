# bufio
包 `bufio` 是封装了 `io` 操作的带一个缓冲区的 `Reader` 和 `Writer`，为我们操作文本 `io` 提供了一些便捷。

## Reader
`Reader` 是封装了 `io.Reader` 接口的一个对象，其本身也实现了 `io.Reader` 接口。

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
