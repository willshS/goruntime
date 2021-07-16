# P
`P` 被称为处理器。

## 1. P的主要字段
```
type p struct {
	id          int32
	status      uint32 // P的状态
	link        puintptr // 链表指针指向下一个P
	schedtick   uint32     // 每次调度++
	syscalltick uint32     // incremented on every system call
	sysmontick  sysmontick // last tick observed by sysmon
	m           muintptr   // P绑定的M
	mcache      *mcache  // P的本地缓存
	pcache      pageCache

  // defer相关
	deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
	deferpoolbuf [5][32]*_defer

	// goroutine id缓存
	goidcache    uint64
	goidcacheend uint64

	// 可运行的goroutine列表
	runqhead uint32  // 头的下标
	runqtail uint32  // 尾的下标
	runq     [256]guintptr // 本地G环形队列

  // 下一个要运行的G，可能为空。继承当前G剩下的时间片如果还有剩余的话。
	runnext guintptr

	// G的空闲队列缓存
	gFree struct {
		gList
		n int32
	}

	sudogcache []*sudog
	sudogbuf   [128]*sudog

	// 内存缓存相关
	mspancache struct {
		len int
		buf [128]*mspan
	}

	runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

	// 调度，是否需要进行调度
	preempt bool
}
```

## 2. P的创建
参考[调度](./sched.md) 2.2小节。

## 3. 总结
关于 `P` 的使用大多都可以在调度相关中看到，为什么一定要有 `P`，调度文档中的官方设计文档也有说明，`P` 中具有内存缓存、时间定时器、可执行协程队列、垃圾回收等各种信息存在。
