# chan
`chan` 是 `golang` 中的通道，是内置的类型。

## 1.使用
`chan` 最主要是用来协程间通信的，有非常非常多的用法。

1. 简单用法，传递信号或数据：
```
func main() {
	c := make(chan int)
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		fmt.Println(<-c)
		wg.Done()
	}()
	c <- 1
	wg.Wait()
}
```

2. 做为互斥锁：（《go专家编程》 p2）
```
var counter int = 0
var ch = make(chan int, 1)

func worker() {
	ch <- 1
	counter++
	<-ch
}
```

## 2. 实现

### 2.1 内置结构
```
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```
