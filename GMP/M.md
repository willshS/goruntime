# M
[GMP的起始](./GMP.md##起始)

## M的主要字段
```
type m struct {
	g0      *g     // 调用栈
	morebuf gobuf  // gobuf arg to morestack

	gsignal       *g           // signal-handling g
	goSigStack    gsignalStack // Go-allocated signal handling stack
	sigmask       sigset       // storage for saved signal mask
	tls           [6]uintptr   // thread-local storage (for x86 extern register)
	mstartfn      func()   // 开始执行的函数
	curg          *g       // 当前正在执行的G
	caughtsig     guintptr // goroutine running during fatal signal
	p             puintptr // 关联的P
	nextp         puintptr // 即将使用的p
	oldp          puintptr // the p that was attached before executing a syscall
	id            int64
	mallocing     int32

	preemptoff    string // if != "", keep curg running on this m
	locks         int32  // 锁 禁止抢占？

	spinning      bool // m 自旋
	blocked       bool // m is blocked on a note
	newSigstack   bool // minit on C thread called sigaltstack
	printlock     int8

	freeWait      uint32 // 如果等于0，可以安全的释放g0并且删除m
	fastrand      [2]uint32 // 随机用的

	traceback     uint8

	doesPark      bool        // non-P running threads: sysmon and newmHandoff never use .park
	park          note
	alllink       *m // on allm
	lockedg       guintptr
	createstack   [32]uintptr // stack that created this thread.
	lockedExt     uint32      // tracking for external LockOSThread
	lockedInt     uint32      // tracking for internal lockOSThread
	nextwaitm     muintptr    // next m waiting for lock
	waitunlockf   func(*g, unsafe.Pointer) bool	// 挂起当前的G要执行的函数 可以为nil
	waitlock      unsafe.Pointer  // 上面函数的第二个参数

	startingtrace bool
	syscalltick   uint32
	freelink      *m // on sched.freem

	// mFixup is used to synchronize OS related m state (credentials etc)
	// use mutex to access.
	mFixup struct {
		lock mutex
		fn   func(bool) bool
	}

	// these are here because they are too large to be on the stack
	// of low-level NOSPLIT functions.
	libcall   libcall
	libcallpc uintptr // for cpu profiler
	libcallsp uintptr
	libcallg  guintptr
	syscall   libcall // stores syscall parameters on windows

	vdsoSP uintptr // SP for traceback while in VDSO call (0 if not in call)
	vdsoPC uintptr // PC for traceback while in VDSO call

	// preemptGen counts the number of completed preemption
	// signals. This is used to detect when a preemption is
	// requested, but fails. Accessed atomically.
	preemptGen uint32

	// Whether this is a pending preemption signal on this M.
	// Accessed atomically.
	signalPending uint32

	dlogPerM

	mOS

	// Up to 10 locks held by this m, maintained by the lock ranking code.
	locksHeldLen int
	locksHeld    [10]heldLockInfo
}
```

## M的创建
以下代码是创建 `M` 的主流程，将 `M` 与系统线程相关联。系统调用 `clone` 在 `sys_linux_amd64.s` 文件中：
```
func newm(fn func(), _p_ *p, id int64) {
  // 分配一个m
	mp := allocm(_p_, fn, id)
  // TODO:是否暂停？
	mp.doesPark = (_p_ != nil)
  // 保存p到即将使用的p
	mp.nextp.set(_p_)
	mp.sigmask = initSigmask
	newm1(mp)
}

// 最终调用clone函数启动一个系统线程，函数为mstart
func newosproc(mp *m) {
	stk := unsafe.Pointer(mp.g0.stack.hi)
	var oset sigset
	sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
	ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
	sigprocmask(_SIG_SETMASK, &oset, nil)
}
```
### M的分配
`M` 的核心逻辑在 `allocm` 中，这里真正创建了 `M` 的结构体以及对部分数据进行初始化：
```
func allocm(_p_ *p, fn func(), id int64) *m {
	_g_ := getg()
	acquirem() // disable GC because it can be called from sysmon
	if _g_.m.p == 0 {
		acquirep(_p_) // temporarily borrow p for mallocs in this function
	}

	// 释放等待释放的M，在释放的同时可以腾出栈的空间
	if sched.freem != nil {
		lock(&sched.lock)
		var newList *m
		for freem := sched.freem; freem != nil; {
			// freeWait标志是否可以释放
			if freem.freeWait != 0 {
				next := freem.freelink
				freem.freelink = newList
				newList = freem
				freem = next
				continue
			}
			// 释放栈
			systemstack(func() {
				stackfree(freem.g0.stack)
			})
			freem = freem.freelink
		}
		// 重新赋值全局等待释放的M链表
		sched.freem = newList
		unlock(&sched.lock)
	}

	mp := new(m)
	mp.mstartfn = fn
	// 做一些通用初始化，比如存到allm，随机种子信号goroutine
	mcommoninit(mp, id)

	// 分配g0栈。不是特殊平台或cgo 8k固定大小的g0栈
	if iscgo || mStackIsSystemAllocated() {
		mp.g0 = malg(-1)
	} else {
		mp.g0 = malg(8192 * sys.StackGuardMultiplier)
	}
	mp.g0.m = mp
	// 借的P 还回去 这是新建的M要用的
	if _p_ == _g_.m.p.ptr() {
		releasep()
	}
	releasem(_g_.m)

	return mp
}
```
