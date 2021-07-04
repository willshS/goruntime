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
	qcount   uint           // 队列中的数据数量
	dataqsiz uint           // 环形队列的大小
	buf      unsafe.Pointer // 环形队列（数组）的指针
	elemsize uint16	// 类型大小
	closed   uint32 // 通道是否关闭
	elemtype *_type // 元素类型
	sendx    uint   // 写入的下标
	recvx    uint   // 读取的下标
	recvq    waitq  // 等待读取的协程队列
	sendq    waitq  // 等待写入的协程队列

	// 锁hchan中的所有字段，以及等待队列中的G的一些字段
	lock mutex
}
```
`waitq` 的实现非常简单，出队和入队，一个双向链表。

### 2.2 chan的创建
调用 `make` 函数创建 `chan`，在经过编译后会被替换为以下函数：
```
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// 检查类型大小
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	// 检查对齐信息 （hchan结构体和类型的对齐）
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}
	// 计算所需内存，并检查内存是否够用
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0:
	  // 当参数size为0 或 元素类型大小为0 的时候，说明不需要额外内存
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// 当元素类型不包含指针的时候，hchan和环形队列所需内存一起分配
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		// 环形队列地址是在hchan后面
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// hchan 和 环形队列内存分开分配
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)
	return c
}
```
以上函数就是创建chan的函数，参数 `t *chantype` 为go语言编译的时候将我们传入的 `chan type` 转为参数 `t` ，这里我们只需要关注 `t.elem` 存储的是通道中的类型信息。此函数重点：  
1. 类型信息检查和内存大小检查
2. 分配内存  
	1. 无需内存的时候不进行分配
	2. 不包含指针的时候内存分配为一块
	3. `chan` 本身内存与环形队列内存分开分配
3. 数据初始化  

TODO:   
1. 类型内存对齐为什么要最大是8？ `hchan` 类型大小为8的倍数？
2. 分配内存包不包含指针不同？ （应与gc有关，后续gc梳理）

### 2.3 chan的写入
与创建相同，使用 `chan <- x` 写入通道的时候，会在编译的时候被替换为 `chansend1`：
```
// 参数 block 标识调用协程是否希望阻塞
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		// 当chan为空的时候
		// 不希望阻塞，返回
		if !block {
			return false
		}
		// 否则挂起调用的协程
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
	// 竞争检查 (根据编译条件，如果有 -race 选项，则 raceenabled 为 true)
	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}

	// 快速路径，检查是否需要阻塞，通道是否关闭，通道是否填满
	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}
	// 锁，并发写入chan的时候需要加锁
	lock(&c.lock)

	if c.closed != 0 {
		// chan已经关闭， panic
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil {
		// 有 goroutine 在等待，直接将数据发送给等待的 g。
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz {
		// 环形队列还有空余空间
		// qp 为环形队列首地址+sendx*elem.size 即元素要放入的内存地址
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		// 注意，这里是有个内存写屏障暂时不管，使用的是memmove将ep 赋值给 qp.
		typedmemmove(c.elemtype, qp, ep)
		// 环形队列相关参数修改
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	// 程序走到这里说明 chan 满了 （关闭和没满的情况已经处理过了）
	// block为false 说明使用的是 select
	if !block {
		unlock(&c.lock)
		return false
	}

	// 生成一个sudog 加入到等待写的链表
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// 等待发送的数据ep 当前的g是gp gp的等待修改为mysg 然后入chan的等待写链表
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)

	// 挂起当前的g. 以下两次调用对于栈进行收缩的时候是无法保证安全的。
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

	// TODO: 这个没看懂。。。
	KeepAlive(ep)

	// 当前g 被唤醒，检查g的waiting 如果不相等，说明这个g的状态已经被破坏了
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	// 这里有个假的唤醒， 关闭的时候可以看到
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// 释放sudog
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```
以上是 `chansend` 函数，实现了一个通道的写入的整个流程。参数说明：  
`c` 表示我们要写入的通道  
`ep` 表示我们要写入通道的数据  
`block` 是根据编译时候通道写入是否在 `select` 中，若不在为 `true`  

-----------------------------------------------------------------
**Notice: 关于 ep 的汇编暂未看懂，因此仅仅为推测**  
关于 `ep` ： go语言是值传递类的语言，因此我们写入通道的数据是被拷贝了一份的：
```
i ：= 1
// 以下是i变量的声明汇编
0x00d9 00217 (main.go:19)       LEAQ    type.int(SB), AX
0x00e0 00224 (main.go:19)       MOVQ    AX, (SP)
0x00e4 00228 (main.go:19)       CALL    runtime.newobject(SB)
0x00e9 00233 (main.go:19)       MOVQ    8(SP), AX
0x00ee 00238 (main.go:19)       MOVQ    AX, "".&i+144(SP)
0x00f6 00246 (main.go:19)       MOVQ    $1, (AX)

c <- i
// 以下是对应写入通道c的汇编
0x0282 00642 (main.go:21)       MOVQ    "".&i+144(SP), AX
0x028a 00650 (main.go:21)       MOVQ    (AX), AX
0x028d 00653 (main.go:21)       MOVQ    AX, ""..autotmp_13+80(SP)
0x0292 00658 (main.go:21)       MOVQ    "".c+88(SP), AX
0x0297 00663 (main.go:21)       MOVQ    AX, (SP)
0x029b 00667 (main.go:21)       LEAQ    ""..autotmp_13+80(SP), AX
0x02a0 00672 (main.go:21)       MOVQ    AX, 8(SP)
0x02a5 00677 (main.go:21)       PCDATA  $1, $4
0x02a5 00677 (main.go:21)       CALL    runtime.chansend1(SB)
```

`MOVQ    AX, ""..autotmp_13+80(SP)` 拷贝数据到匿名变量  
当我们传值的时候无论 `ep` 是否是临时变量又拷贝了一份，在写入通道的时候，我们一定会使用memmove再次拷贝一次。

-----------------------------------------------------------------

TODO:  
1. 栈收缩对于当前变量g的影响 以及ep 可能为栈变量的指针
2. g调度和sudug相关  

关于 `block` ： 这个参数决定了当无法写入通道的时候（通道为 `nil` 或已经满了），当前协程是挂起还是直接返回 `false`  

**通道写入流程如下**：  
1. `chan` 为空，非阻塞模式直接返回，阻塞模式挂起`G`。 `chan` 未关闭且环形队列已满，非阻塞模式直接返回。
2. `chan` 已经关闭，调用 `panic`
3. 此时有协程等待读取，直接将数据发送给等待链表的第一个协程
4. 环形队列还有空间，数据放入环形队列
5. 环形队列已满，挂起当前 `G`

### 2.4 chan的读取
当我们使用 `tmp := <- c` 的时候，编译器会替换为调用 `chanrecv1` 函数。同理，如果是 `tmp, ok := <- c` 的时候，会调用 `chanrecv2`。核心都是下面的函数：
```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if c == nil {
		// 这里跟发送一样的处理逻辑
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if !block && empty(c) {
		// 这里判断如果为空再去判断是否关闭，否则一起判断如果关闭的话，会造成锁的竞争。
		// 这里的注释非常难以理解。包括为何使用原子操作等。（TODO:）
		if atomic.Load(&c.closed) == 0 {
			// 关闭操作是不可逆的，因此这里如果没有关闭，那么上面的判断时肯定也是没有关闭的。同理，下面的一定是关闭的时候发生。
			return
		}
		// 这里再次检查是否为空，因为在上面判空之后判断关闭之前可能又有数据到达
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)
	// 已经关闭 并且 环形队列为空
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			// memclr 参数
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	if sg := c.sendq.dequeue(); sg != nil {
		// 有等待发送的G 这里分为两种情况，一种是环形队列为空，直接将阻塞的sg的数据拷贝到ep。
		// 一种是环形队列是满的，从环形队列队首元素取出拷贝到ep，将sg的数据进入环形队列，唤醒sg
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 这里就是环形队列不为空也不为满的情况，取 qp 拷贝给 ep，清除qp。
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	// 运行到这里就是环形队列为空，并且没有等待发送数据的sg
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 与 chansend 相同，挂起当前的G
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// close的时候会将param赋值为nil
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```
读取流程与写入流程大同小异，对于快速返回有一个优化比较难以理解。我理解的可能也不太对。
**通道写入流程如下**：  
1. `chan` 为空，非阻塞模式直接返回，阻塞模式挂起`G`。 `chan` 未关闭且环形队列为空，非阻塞模式直接返回。
2. `chan` 已经关闭，且环形队列为空，清空类型信息并返回
3. 此时有协程等待发送，根据环形队列为空还是为满，分情况处理。
4. 环形队列还有数据，数据拷贝给 `ep`
5. 环形队列为空且无协程等待发送，挂起当前 `G`

**拷贝次数：无缓冲区或有相关阻塞的sg时，是将 `sg` 中的数据直接拷贝到相应的参数 `ep`。当缓冲区有数据或者有空闲空间，发送会将数据拷贝到缓冲区，读取会将缓冲区的数据拷贝到接收的 `ep`。 如果不算发送或接收的临时变量的拷贝的话，我们至少拷贝一次，一般是两次。因此不要用通道传递庞大的结构体，改为传递结构体的指针（这样拷贝的就是结构体的指针了，4个字节或者8个字节）**

### 2.5 chan的关闭
```
func closechan(c *hchan) {
	// 关闭空的 直接 panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	// 关闭已经关闭的 panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1

	var glist gList

	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		// 注意，这里将阻塞发送的param赋值为nil 因此发送者被唤醒后会检查此参数而导致panic
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// 唤醒所有的阻塞的发送者或者接收者
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```
一个小的注意点，参数 `gp.param` 当有数据唤醒阻塞的协程时， `param` 为阻塞时生成的 `sudog` 的指针。关闭时将所有阻塞的协程的 `param` 赋值为 `nil`

## 3 总结
从上面流程可以看到，通道无缓冲区和有缓冲区的处理逻辑是一模一样的，仅仅是环形队列的大小为0。  
1. 对于 `nil` 的通道，读写都会阻塞
2. `panic` 的发生：
	1. 写关闭的通道
	2. 关闭已经关闭的通道
	3. 关闭 `nil` 的通道
	4. 关闭通道，阻塞等待的发送者

### 3.1 一点小补充
`select` 编译相关的函数：
```
// compiler implements
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}

// compiler implements
//
//	select {
//	case v = <-c:
//		... foo
//	default:
//		... bar
//	}
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

// compiler implements
//
//	select {
//	case v, ok = <-c:
//		... foo
//	default:
//		... bar
//	}
func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
	// TODO(khr): just return 2 values from this function, now that it is in Go.
	selected, *received = chanrecv(c, elem, false)
	return
}
```
传入的 `block` 参数都为 `false`。

### 3.2 两点小补充
单向通道。单向通道仅仅是对通道的使用加以限定，让编译器做一个类型检查。
```
// 《go专家编程》 P11
// 只读限定
func readChan(chanName <-chan int) {
	<- chanName
}
// 只写限定
func writeChan(chanName chan<- int) {
	chanName <- 1
}

func main() {
	var mychan = make(chan int, 1)
	wirteChan(mychan)
	readChan(mychan)
}
```
创建一个单向通道是毫无意义的，不具有通道功能了。
```
test := make(chan<- int)
test <- 1
```
