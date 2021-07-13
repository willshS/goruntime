# G
`goroutine` 是 `go` 语言并发的基石，从本质来讲，它是用户级线程。通过实现调度算法，与系统级线程进行交互，从而使协程可以在线程上运行。  

## G的主要字段
`g` 保存了 `goroutine` 相关信息，注重到的是堆栈和调度信息以及相关状态。从这里看协程其实就是一组数据，是一个抽象的概念。因此对协程的执行，其实就是基于 `g` 结构体的执行相关指令。协程的调度，与线程的调度是一样的，恢复协程的栈，在栈上继续执行指令。
```
type g struct {
	// 栈参数 stack 描述实际的栈内存 [stack.lo, stack.hi)
	stack       stack
  // 一个栈指针，值为StackPreempt可以触发抢占
	stackguard0 uintptr
  // g所在的m，可能为nil
	m            *m
  // 协程调度需要保存的信息
	sched        gobuf
  // chan里面见过了，唤醒的时候传递的参数
	param        unsafe.Pointer // passed parameter on wakeup
  // G的状态
  atomicstatus uint32
	// G的唯一标识
	goid         int64
	schedlink    guintptr	// 调度链表
  // block相关
	waitsince    int64      // approx time when the g become blocked
	waitreason   waitReason // if status==Gwaiting
  // 调度相关
	preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
	preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
	preemptShrink bool // shrink stack at synchronous safe point
  // 阻塞到chan上了
	parkingOnChan uint8
	gopc           uintptr         // 创建这个协程的协程的pc
	startpc        uintptr         // 这个协程的开始程序计数器
	waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
	timer          *timer         // cached timer for time.Sleep
	selectDone     uint32         // are we participating in a select and did someone win the race?
}
```

## G的创建
创建 `goroutine` 是通过调用 `newProc`。即我们使用 `go func` 创建新协程。
```
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
	pc := getcallerpc()
  // 在系统栈上创建新的协程
	systemstack(func() {
    // 创建一个新的G
		newg := newproc1(fn, argp, siz, gp, pc)

		_p_ := getg().m.p.ptr()
    // 放入P的可运行队列
		runqput(_p_, newg, true)

		if mainStarted {
			wakep()
		}
	})
}
```
创建新的 `goroutine` 函数如下（省略了很多代码）：
```
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
	_g_ := getg()

	acquirem() // disable preemption because it can be holding p in a local var
	siz := narg
	siz = (siz + 7) &^ 7

  // 复用一个空闲的G
	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_)
	if newg == nil {
    // 没有空闲的，分配一个 _StackMin = 2k
		newg = malg(_StackMin)
    // 修改新创建的G的状态
		casgstatus(newg, _Gidle, _Gdead)
    // 加入全局gs
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}

	totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
	totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
	sp := newg.stack.hi - totalSize
	spArg := sp

	if narg > 0 {
    // 拷贝参数到栈顶
		memmove(unsafe.Pointer(spArg), argp, uintptr(narg))
	}
  // 初始化调度信息
	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	newg.sched.sp = sp
	newg.stktopsp = sp
  // 初始化程序计数器为goexit
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
  // 这里继续初始化：sched.sp = sched.pc && sched.pc = fn
	gostartcallfn(&newg.sched, fn)
	newg.gopc = callerpc
	newg.startpc = fn.fn
  // 修改G的状态为可运行
	casgstatus(newg, _Gdead, _Grunnable)

  // 生成G的id
	if _p_.goidcache == _p_.goidcacheend {
		_p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
		_p_.goidcache -= _GoidCacheBatch - 1
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
	}
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
	releasem(_g_.m)

	return newg
}
```
