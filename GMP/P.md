# P
`P` 被成为调度器。

## P的主要字段
```
type p struct {
	id          int32
	status      uint32 // P的状态
	link        puintptr // 链表指针指向下一个P
	schedtick   uint32     // incremented on every scheduler call
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
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr

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
