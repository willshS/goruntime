# sched
关于GMP，网上特别多博客资料。  
[写得非常好的总结](https://github.com/fuweid/notes/blob/master/golang/goroutine_scheduler_overview.md)  
[官方的一些说明](https://github.com/golang/go/blob/048c9cfaacb6fe7ac342b0acd8ca8322b6c49508/src/runtime/HACKING.md)  
[官方的设计文档](https://golang.org/s/go11sched)  
[draveness大佬写的书](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)

## 1. 调度的全局参数
```
type schedt struct {
	goidgen   uint64 // 原子访问，goroutine的id全局生成器
	lastpoll  uint64 // time of last network poll, 0 if currently polling
	pollUntil uint64 // time to which current poll is sleeping

	lock mutex

	midle        muintptr // 空闲M链表
	nmidle       int32    // 空闲M数量
	nmidlelocked int32    // number of locked m's waiting for work
	mnext        int64    // number of m's that have been created and next M ID
	maxmcount    int32    // maximum number of m's allowed (or die)
	nmsys        int32    // number of system m's not counted for deadlock
	nmfreed      int64    // cumulative number of freed m's

	ngsys uint32 // number of system goroutines; updated atomically

	pidle      puintptr // 空闲P链表
	npidle     uint32	// 空间P数量
	nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

	// 全局G 队列
	runq     gQueue
	runqsize int32

	// disable controls selective disabling of the scheduler.
	//
	// Use schedEnableUser to control this.
	//
	// disable is protected by sched.lock.
	disable struct {
		// user disables scheduling of user goroutines.
		user     bool
		runnable gQueue // pending runnable Gs
		n        int32  // length of runnable
	}

	// 全局空闲G的缓存
	gFree struct {
		lock    mutex
		stack   gList // Gs with stacks
		noStack gList // Gs without stacks
		n       int32
	}

	// Central cache of sudog structs.
	sudoglock  mutex
	sudogcache *sudog

	// Central pool of available defer structs of different sizes.
	deferlock mutex
	deferpool [5]*_defer

	// 等待释放的M链表
	freem *m

	gcwaiting  uint32 // gc is waiting to run
	stopwait   int32
	stopnote   note
	sysmonwait uint32
	sysmonnote note

	sysmonStarting uint32

	// safepointFn should be called on each P at the next GC
	// safepoint if p.runSafePointFn is set.
	safePointFn   func(*p)
	safePointWait int32
	safePointNote note

	procresizetime int64 // nanotime() of last change to gomaxprocs
	totaltime      int64 // ∫gomaxprocs dt up to procresizetime

	// sysmonlock protects sysmon's actions on the runtime.
	//
	// Acquire and hold this mutex to block sysmon from interacting
	// with the rest of the runtime.
	sysmonlock mutex
}
```

## 2. 程序的启动与退出

### 2.1 起始
二进制可执行文件在 `LINUX` 上为 `elf` 格式。可以使用命令 `readelf` 读取可执行文件的相关信息。最终分析可以获取程序的入口为 `_rt0_amd64_linux.s` 的 `_rt0_amd64_linux` ：
```
TEXT _rt0_arm64_linux(SB),NOSPLIT|NOFRAME,$0
	MOVD	0(RSP), R0	// argc
	ADD	$8, RSP, R1	// argv
	BL	main(SB)
```
兜兜转转到真正的起始函数-- `asm_arm64.s` 的 `runtime·rt0_go` 汇编函数：
```
// set the per-goroutine and per-mach "registers"
MOVD	$runtime·m0(SB), R0

// save m->g0 = g0
MOVD	g, m_g0(R0)
// save m0 to g0->m
MOVD	R0, g_m(g)

BL	runtime·check(SB)

MOVW	8(RSP), R0	// copy argc
MOVW	R0, -8(RSP)
MOVD	16(RSP), R0		// copy argv
MOVD	R0, 0(RSP)
BL	runtime·args(SB)
BL	runtime·osinit(SB)
BL	runtime·schedinit(SB)

// create a new goroutine to start program
MOVD	$runtime·mainPC(SB), R0		// entry
MOVD	RSP, R7
MOVD.W	$0, -8(R7)
MOVD.W	R0, -8(R7)
MOVD.W	$0, -8(R7)
MOVD.W	$0, -8(R7)
MOVD	R7, RSP
BL	runtime·newproc(SB)
ADD	$32, RSP

// start this M
BL	runtime·mstart(SB)
```
从上面可以看到主要步骤为： `g0` 和 `m0` 的关联、参数处理、 `osinit` 系统相关初始化、`schedinit` 调度初始化、`newproc` 创建入口为 `main` 的新的协程，`mstart` 开始执行。程序开始运行，操作系统必然会创建一个进程和一个线程，这里的 `m0` 就是主线程，`g0` 是与 `m0` 关联的主协程，所有初始化操作都是在这个线程和协程做的。`g0` 是一个特殊协程，为了调度在 `M` 上运行的协程。其使用的堆栈也即“系统堆栈”（固定大小8k）。

### 2.2 初始化
`schedinit` 函数是初始化，不仅仅是调度初始化，还有关于垃圾回收、命令行参数、环境等等。初始化调度相关的就是对 `P` 的初始化。
```
// 根据gomaxprocs 初始化相应的P 此函数也可能在程序运行过程中调用
func procresize(nprocs int32) *p {
	......

	if nprocs > int32(len(allp)) {
		// 初始化全局变量allp
		lock(&allpLock)
		if nprocs <= int32(cap(allp)) {
			allp = allp[:nprocs]
		} else {
			nallp := make([]*p, nprocs)
			// Copy everything up to allp's cap so we
			// never lose old allocated Ps.
			copy(nallp, allp[:cap(allp)])
			allp = nallp
		}
		unlock(&allpLock)
	}

	// initialize new P's
	for i := old; i < nprocs; i++ {
		// 初始化P
		pp := allp[i]
		if pp == nil {
			pp = new(p)
		}
		pp.init(i)
		atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
	}

	// allp[0] 给 m0
	_g_ := getg()
	if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
		// continue to use the current P
		_g_.m.p.ptr().status = _Prunning
		_g_.m.p.ptr().mcache.prepareForSweep()
	} else {
		_g_.m.p = 0
		p := allp[0]
		p.m = 0
		p.status = _Pidle
		acquirep(p)
		if trace.enabled {
			traceGoStart()
		}
	}

	// 释放多余的P
	for i := nprocs; i < old; i++ {
		p := allp[i]
		p.destroy()
		// can't free P itself because it can be referenced by an M in syscall
	}

	// 剩余空闲的P 串成链表返回
	var runnablePs *p
	for i := nprocs - 1; i >= 0; i-- {
		p := allp[i]
		if _g_.m.p.ptr() == p {
			continue
		}
		p.status = _Pidle
		if runqempty(p) {
			// 没任务的P 放入全局空闲的P队列
			pidleput(p)
		} else {
			// 有任务的返回出去 （初始化调用这里的时候，一定没有P存在本地的G）
			p.m.set(mget())
			p.link.set(runnablePs)
			runnablePs = p
		}
	}
	stealOrder.reset(uint32(nprocs))
	var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
	atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
	return runnablePs
}
```

### 2.3 调度的主流程
`newproc` 创建协程后，就要开始调度，调度开始为 `mstart`，此函数也是创建系统线程所传入的函数地址。 `mstart` 初始化栈边界以及栈守卫（抢占相关），然后调用 `mstart1`，此函数先调用 `m.mstartfn`。之后调用 `schedule` 开始调度：
```
func schedule() {
	_g_ := getg()

	if _g_.m.lockedg != 0 {
		stoplockedm()
		// 如果绑定了g，直接运行绑定的g
		execute(_g_.m.lockedg.ptr(), false) // Never returns.
	}

top:
	pp := _g_.m.p.ptr()
	pp.preempt = false

	var gp *g
	var inheritTime bool

	if gp == nil {
		// 根据P的schedtick，判断是否先去全局队列拿可执行的G。
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		// 本地队列找一个G
		gp, inheritTime = runqget(_g_.m.p.ptr())
	}
	if gp == nil {
		// 阻塞找G
		gp, inheritTime = findrunnable() // blocks until work is available
	}

	// 如果M正在自旋，取消，如果有空闲P，新建一个M自旋
	if _g_.m.spinning {
		resetspinning()
	}

	// 禁止调度用户协程 或 无法调度系统协程
	if sched.disable.user && !schedEnabled(gp) {
		lock(&sched.lock)
		if schedEnabled(gp) {
			unlock(&sched.lock)
		} else {
			sched.disable.runnable.pushBack(gp)
			sched.disable.n++
			unlock(&sched.lock)
			goto top
		}
	}

	if gp.lockedm != 0 {
		// Hands off own p to the locked m,
		// then blocks waiting for a new p.
		startlockedm(gp)
		goto top
	}

	// 执行
	execute(gp, inheritTime)
}
```

### 2.4 执行
调度最终会将一个可运行的 `G` 进行执行。
```
func execute(gp *g, inheritTime bool) {
	_g_ := getg()

	// gp分配给m的当前g
	_g_.m.curg = gp
	gp.m = _g_.m
	// 修改G的状态为运行中
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		// 增加调度时间片
		_g_.m.p.ptr().schedtick++
	}

	// Check whether the profiler needs to be turned on or off.
	hz := sched.profilehz
	if _g_.m.profilehz != hz {
		setThreadCPUProfiler(hz)
	}
	// 执行
	gogo(&gp.sched)
}
```
`gogo` 是用汇编实现的，其主要实现就是将 `G.sched` 中的数据取出并执行 `G.sched.pc` 指令，进入 `goroutine` 的函数。并将 `goexit` 作为执行完后要执行的函数。（这里推荐draveness大佬的对于函数调用汇编的分析以及gogo汇编的分析，第4个链接）

### 2.5 main函数的开始
`m0` 拥有 `P` 之后，就会调用 `newproc` 创建协程，这个协程的地址为 `runtime.main`，此函数为位于 `proc.go` 的 `main` 函数。在这个函数中会调用我们在 `main` 包中的我们自己的 `main` 函数。
```
func main() {
	g := getg()

	// main启动
	mainStarted = true

	if GOARCH != "wasm" {
		// 启动sysmon
		atomic.Store(&sched.sysmonStarting, 1)
		systemstack(func() {
			newm(sysmon, nil, -1)
		})
	}

	// 锁线程 TODO：补充
	lockOSThread()
	// 只有主线程m0 可以运行main
	if g.m != &m0 {
		throw("runtime.main not on m0")
	}
	m0.doesPark = true
	// 包初始化相关
	doInit(&runtime_inittask) // Must be before defer.

	// Defer unlock so that runtime.Goexit during init does the unlock too.
	needUnlock := true
	defer func() {
		if needUnlock {
			unlockOSThread()
		}
	}()
	// 打开gc
	gcenable()

	main_init_done = make(chan bool)
	doInit(&main_inittask)
	close(main_init_done)
	// 释放锁线程
	needUnlock = false
	unlockOSThread()

	// main.main
	fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()
	// 退出进程
	exit(0)
	for {
		var x *int32
		*x = 0
	}
}
```

### 2.6 总结
以上是我们的程序整个执行的流程。简略的介绍了主体从开始到结束，以及调度的主要流程。

## 3. 调度
这一节介绍调度普通 `G` 的流程，调度的主体是协程，因此调度最重要的是对可执行协程的选择。

### 3.1 可运行G的存储
首先我们要将可运行的协程进行存储，以便调度的时候进行选择。参考 `G` 的创建，当成功创建一个协程后，我们调用 `runqput` 来存储可运行的协程：
```
func runqput(_p_ *p, gp *g, next bool) {
	// 如果下一个就要执行（与上一个正在执行的共享时间片），
	if next {
	retryNext:
		oldnext := _p_.runnext
		// 原子写入P.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		// 以前的是空，放入成功
		if oldnext == 0 {
			return
		}
		// 原来的next 赋值给gp 继续走放入队列的过程
		gp = oldnext.ptr()
	}

retry:
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
	// 如果有空位
	if t-h < uint32(len(_p_.runq)) {
		// 放入队列的末尾
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
	// 没有空位放入全局队列
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// 重试
	goto retry
}
```
当本地队列满了的时候，我们需要调用 `runqputslow` 尝试放入全局队列：
```
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
	var batch [len(_p_.runq)/2 + 1]*g

	// 抓个快照
	n := t - h
	n = n / 2
	if n != uint32(len(_p_.runq)/2) {
		throw("runqputslow: queue is not full")
	}
	//将本地队列的前1/2 放入batch
	for i := uint32(0); i < n; i++ {
		batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))].ptr()
	}
	// 修改本地队列的头下标
	if !atomic.CasRel(&_p_.runqhead, h, h+n) { // cas-release, commits consume
		return false
	}
	// 正主放入末尾
	batch[n] = gp

	// Link the goroutines.
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}
	var q gQueue
	q.head.set(batch[0])
	q.tail.set(batch[n])

	// 全局大锁 将队列q放入全局
	lock(&sched.lock)
	// 这里就是全局队列的链表操作了，不贴了
	globrunqputbatch(&q, int32(n+1))
	unlock(&sched.lock)
	return true
}
```
可以看到新创建的可运行 `G` 先尝试放入本地队列，如果本地队列已满，将本地队列的前1/2以及新创建的协程放入全局队列。
### 3.2 可运行G的取出
从2.3调度主流程可以看到一个大概的取的过程:
1. 根据全局调度器的 `schedtick%61` 判断是否先去全局队列取
2. 调用 `runqget` 去本地队列取，先从 `runnext` 取，不存在再去本地队列中取
3. 前两步还没取到，调用 `findrunnable` 找一个可运行的协程
	1. 再次查找本地队列和全局队列
	2. 从网络轮询器中查看是否有可运行的
	3. 随机去所有P里面偷，偷的顺序是随机的，最多尝试4次（如果全部失败，最后还会检查一次）

**TODO：上面的过程极其复杂，还涉及到了gc的工作，timer相关，以及网络轮询器。后面再一一分析。**
```
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()
top:
	_p_ := _g_.m.p.ptr()
	now, pollUntil, _ := checkTimers(_p_, 0)

	// 再次尝试本地队列
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

	// 从全局取部分G，并放入本地队列 （取的数量为min(sched.runqsize/gomaxprocs + 1, (len(_p_.runq))/2)）
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}

	// 这里是一个优化，先从网络轮询器里面查看是否有等待运行的G TODO:netpoll
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}

	// 从其它P那里偷
	procs := uint32(gomaxprocs)
	ranTimer := false
	// 如果自旋的M大于等于忙碌的P，阻塞，减小cpu的消耗
	if !_g_.m.spinning && 2*atomic.Load(&sched.nmspinning) >= procs-atomic.Load(&sched.npidle) {
		goto stop
	}
	// 当前M自旋
	if !_g_.m.spinning {
		_g_.m.spinning = true
		atomic.Xadd(&sched.nmspinning, 1)
	}
	const stealTries = 4
	for i := 0; i < stealTries; i++ {
		stealTimersOrRunNextG := i == stealTries-1
		// 随机选取一个P
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			p2 := allp[enum.position()]
			if _p_ == p2 {
				continue
			}

			// TODO: timer
			if stealTimersOrRunNextG && timerpMask.read(enum.position()) {
				tnow, w, ran := checkTimers(p2, now)
				now = tnow
				if w != 0 && (pollUntil == 0 || w < pollUntil) {
					pollUntil = w
				}
				if ran {
					// Running the timers may have
					// made an arbitrary number of G's
					// ready and added them to this P's
					// local run queue. That invalidates
					// the assumption of runqsteal
					// that is always has room to add
					// stolen G's. So check now if there
					// is a local G to run.
					if gp, inheritTime := runqget(_p_); gp != nil {
						return gp, inheritTime
					}
					ranTimer = true
				}
			}

			// 检查P2的状态，空闲的话就不要去偷了，它肯定没东西
			if !idlepMask.read(enum.position()) {
				// 偷：这个函数就是检查p2的可执行协程队列，然后偷前1/2过来（必须小于p2本地队列的一半）。
				if gp := runqsteal(_p_, p2, stealTimersOrRunNextG); gp != nil {
					return gp, false
				}
			}
		}
	}
	if ranTimer {
		// Running a timer may have made some goroutine ready.
		goto top
	}

stop:
	// 保存P的快照
	allpSnapshot := allp
	idlepMaskSnapshot := idlepMask
	timerpMaskSnapshot := timerpMask
	// 再次尝试
	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false
	}
	// 释放P
	if releasep() != _p_ {
		throw("findrunnable: wrong p")
	}
	// 放入P的空闲队列
	pidleput(_p_)
	unlock(&sched.lock)

	// 自旋状态解除
	wasSpinning := _g_.m.spinning
	if _g_.m.spinning {
		_g_.m.spinning = false
		if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
			throw("findrunnable: negative nmspinning")
		}
	}

	// 再次检查所有的P
	for id, _p_ := range allpSnapshot {
		if !idlepMaskSnapshot.read(uint32(id)) && !runqempty(_p_) {
			lock(&sched.lock)
			_p_ = pidleget()
			unlock(&sched.lock)
			if _p_ != nil {
				// 有任务再次获取一个P，再去尝试一次获取
				acquirep(_p_)
				if wasSpinning {
					_g_.m.spinning = true
					atomic.Xadd(&sched.nmspinning, 1)
				}
				goto top
			}
			break
		}
	}

	// 跟上面一样，看有没有timer？ TODO:
	for id, _p_ := range allpSnapshot {
		if timerpMaskSnapshot.read(uint32(id)) {
			w := nobarrierWakeTime(_p_)
			if w != 0 && (pollUntil == 0 || w < pollUntil) {
				pollUntil = w
			}
		}
	}
	if pollUntil != 0 {
		if now == 0 {
			now = nanotime()
		}
		delta = pollUntil - now
		if delta < 0 {
			delta = 0
		}
	}

	// 再次检查gc的工作和网络轮询器的工作

	// 停止M
	stopm()
	goto top
}
```  
可以看到整个调度的核心就是找可运行的协程。
### 3.3 可运行G的执行完毕
协程执行完毕后，会调用以下函数：
```
func goexit0(gp *g) {
	_g_ := getg()
	// G状态
	casgstatus(gp, _Grunning, _Gdead)
	if isSystemGoroutine(gp, false) {
		atomic.Xadd(&sched.ngsys, -1)
	}
	// reset g的所有数据
	gp.m = nil
	locked := gp.lockedm != 0
	gp.lockedm = 0
	_g_.m.lockedg = 0
	gp.preemptStop = false
	gp.paniconfault = false
	gp._defer = nil // should be true already but just in case.
	gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
	gp.writebuf = nil
	gp.waitreason = 0
	gp.param = nil
	gp.labels = nil
	gp.timer = nil

	dropg()
	// 放入本地空闲G的缓存池
	gfput(_g_.m.p.ptr(), gp)
	// 调度下一个
	schedule()
}
```
协程执行结束，主要是各种状态的重置，以及继续调度。

## 4. 其它触发调度的方式
调度完全通过函数 `schedule()` 来进行，除了上面介绍的类似于线程池线程开始和任务执行结束开始调度外，我们只需要看看哪里调用了调度函数就知道还有什么地方会进行调度

### 4.1 协程主动挂起导致的调度
当协程等待某些资源的时候可以进行主动挂起，有其它协程满足资源时需要被唤醒

#### 4.1.1 主动挂起
我们知道协程写入通道的时候，如果通道已满（包括无缓冲通道）或 通道为 `nil`，则协程会阻塞，参考[chan源码解析](../datastruct/chan/chan.md)。此时需要阻塞的协程调用 `gopark` 函数进入阻塞等待：
```
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	// m挂起G相关变量
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason

	releasem(mp)
	// can't do anything that might move the G between Ms here.
	mcall(park_m)
}

func park_m(gp *g) {
	_g_ := getg()
	// 修改要挂起的G的状态为等待
	casgstatus(gp, _Grunning, _Gwaiting)
	// 与M的curg解除
	dropg()
	// 调用挂起G 传入的函数 （函数和参数都是要挂起的G传入的）
	if fn := _g_.m.waitunlockf; fn != nil {
		ok := fn(gp, _g_.m.waitlock)
		_g_.m.waitunlockf = nil
		_g_.m.waitlock = nil
		if !ok {
			// 返回false 可以重新运行
			casgstatus(gp, _Gwaiting, _Grunnable)
			execute(gp, true) // Schedule it back, never returns.
		}
	}
	// 重新调度
	schedule()
}
```
通过 `waitunlockf` 函数，可以很轻松的对不同业务做处理，比如 `chansend` 阻塞到已满缓冲区的 `chan` 时，此时要读取数据，所以已经对 `chan` 进行了加锁操作，那么可以在 `waitunlockf` 函数中进行解锁操作，不影响其它对这个 `chan` 操作的协程。而且可以根据 `waitunlockf` 函数的返回值来确定是否要真的挂起重新调度。

#### 4.1.2 唤醒
协程主动挂起后，状态为 `_Gwaiting`，此状态表示协程为阻塞状态。等待条件触发时，需要调用 `goready` 来进行唤醒，核心函数为 `ready` 函数：
```
// Mark gp ready to run.
func ready(gp *g, traceskip int, next bool) {
	status := readgstatus(gp)

	// Mark runnable.
	_g_ := getg()
	mp := acquirem() // disable preemption because it can be holding p in a local var
	if status&^_Gscan != _Gwaiting {
		dumpgstatus(gp)
		throw("bad g->status in ready")
	}

	// 修改协程状态 并放入本地队列
	casgstatus(gp, _Gwaiting, _Grunnable)
	runqput(_g_.m.p.ptr(), gp, next)
	wakep()
	releasem(mp)
}
```

### 4.2 协程主动让出CPU
调用 `Gosched` 函数可以让当前协程让出CPU，但是不进行挂起。 `Gosched` 的核心非常简单：
```
func goschedImpl(gp *g) {
	status := readgstatus(gp)
	if status&^_Gscan != _Grunning {
		dumpgstatus(gp)
		throw("bad g status")
	}
	// 修改状态为可运行
	casgstatus(gp, _Grunning, _Grunnable)
	// 解绑curg
	dropg()
	lock(&sched.lock)
	// 放入全局可运行队列
	globrunqput(gp)
	unlock(&sched.lock)
	// 调度
	schedule()
}
```
需要注意的是，协程被放入了全局可运行队列等待被调度，而非本地的可运行队列。  
还有一个与 `Gosched` 类似的函数 `goyield`，两个函数基本相同，只不过后者将协程放入了本地的可运行队列。这里就不贴代码了。

### 4.3 系统调用调度
### 4.4 抢占调度
