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
	lastpoll  uint64 // 上次网络轮询的时间，正在进行网络轮询的时候是0
	pollUntil uint64 // time to which current poll is sleeping

	lock mutex

	midle        muintptr // 空闲M链表
	nmidle       int32    // 空闲M数量
	nmidlelocked int32    // 被绑定的空闲的M的数量
	mnext        int64    // m的数量，也是下一个m的id的
	maxmcount    int32    // 最大允许的m的数量
	nmsys        int32    // 系统的M的数量
	nmfreed      int64    // 释放的M的数量

	ngsys uint32 // 系统协程的数量 isSystemGoroutine 判断

	pidle      puintptr // 空闲P链表
	npidle     uint32	// 空间P数量
	nmspinning uint32 // 自旋M的数量

	// 全局G 队列
	runq     gQueue
	runqsize int32

	// 有些情况下需要禁止调度
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

	// sudog的缓存
	sudoglock  mutex
	sudogcache *sudog

	// Central pool of available defer structs of different sizes.
	deferlock mutex
	deferpool [5]*_defer

	// 等待释放的M链表
	freem *m

	gcwaiting  uint32 // 1 gc等待运行
	stopwait   int32  // 等待停止P的数量
	stopnote   note	// note 当所有P停止，用来通知可以gc的
	sysmonwait uint32	// sysmon的状态
	sysmonnote note // sysmon的note

	sysmonStarting uint32

	// safepointFn should be called on each P at the next GC
	// safepoint if p.runSafePointFn is set. TODO:安全点
	safePointFn   func(*p)
	safePointWait int32
	safePointNote note

	procresizetime int64 // 上次改变P数量的时间
	totaltime      int64 // ∫gomaxprocs dt up to procresizetime

	sysmonlock mutex	//sysmon的锁
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
	// 重置g的调度状态
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
				// 偷并不需要加锁，类似于环形无锁队列，先抓取快照，然后计算拷贝要偷的任务，之后cas确定没有竞争。
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

### 3.4 sysmon监控
`runtime.main` 中我们启动了 `sysmon`，此函数为监控函数：
```
func sysmon() {
	lock(&sched.lock)
	sched.nmsys++
	checkdead()
	unlock(&sched.lock)

	atomic.Store(&sched.sysmonStarting, 0)

	lasttrace := int64(0)
	idle := 0 // how many cycles in succession we had not wokeup somebody
	delay := uint32(0)

	for {
		if idle == 0 { // start with 20us sleep...
			delay = 20
		} else if idle > 50 { // start doubling the sleep after 1ms...
			delay *= 2
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000
		}
		// 休眠一会
		usleep(delay)
		mDoFixup()

		now := nanotime()
		if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {
			lock(&sched.lock)
			// 如果正在gc，或者所有P都是空闲的
			if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
				syscallWake := false
				// 检查所有P的timer
				next, _ := timeSleepUntil()
				if next > now {
					// 修改sysmon的状态
					atomic.Store(&sched.sysmonwait, 1)
					unlock(&sched.lock)
					// 睡眠时间计算
					sleep := forcegcperiod / 2
					if next-now < sleep {
						sleep = next - now
					}
					// 系统调用 futex 休眠
					syscallWake = notetsleep(&sched.sysmonnote, sleep)
					mDoFixup()
					lock(&sched.lock)
					// 休眠结束 状态改回来
					atomic.Store(&sched.sysmonwait, 0)
					noteclear(&sched.sysmonnote)
				}
				// 这里表示被唤醒了 还是休眠时间到了
				if syscallWake {
					idle = 0
					delay = 20
				}
			}
			unlock(&sched.lock)
		}

		lock(&sched.sysmonlock)
		now = nanotime()

		// 调用网络轮询器
		lastpoll := int64(atomic.Load64(&sched.lastpoll))
		if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
			atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
			list := netpoll(0) // non-blocking - returns list of goroutines
			if !list.empty() {
				// 对于触发网络事件的协程，放入可运行队列 （sysmon运行没有P，这里放入全局可运行队列）
				incidlelocked(-1)
				injectglist(&list)
				incidlelocked(1)
			}
		}
		mDoFixup()
		// 检查所有P的状态
		if retake(now) != 0 {
			idle = 0
		} else {
			idle++
		}
		unlock(&sched.sysmonlock)
	}
}
```

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
#### 4.3.1 进入系统调用
在调用系统函数时，汇编实现的 `syscall` 会先调用 `entersyscall`，表示进入系统调用：
```
func reentersyscall(pc, sp uintptr) {
	_g_ := getg()
	_g_.m.locks++
	// g的状态改为可抢占
	_g_.stackguard0 = stackPreempt
	_g_.throwsplit = true

	// 保存调用系统函数的调用者的pc和sp
	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
	// 修改G的状态为系统调用
	casgstatus(_g_, _Grunning, _Gsyscall)

	if atomic.Load(&sched.sysmonwait) != 0 {
		// 如果sysmon此时在休眠，把它唤醒
		systemstack(entersyscall_sysmon)
		save(pc, sp)
	}
	// m的系统调用次数 = p的系统调用次数
	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick
	_g_.sysblocktraced = true
	// 把P 给M 的oldp 藕断丝连的解绑
	pp := _g_.m.p.ptr()
	pp.m = 0
	_g_.m.oldp.set(pp)
	_g_.m.p = 0
	// P状态改为系统调用
	atomic.Store(&pp.status, _Psyscall)
	if sched.gcwaiting != 0 {
		// 如果GC正在等的操作
		systemstack(entersyscall_gcwait)
		save(pc, sp)
	}

	_g_.m.locks--
}
```
一个协程进入系统调用，我们并没有直接将 `P` 与 `M` 直接解绑，而是半解绑的状态。还有一个系统调用前调用的函数为 `entersyscallblock` ，它与上面函数类似，只是在确定此系统调用会阻塞一段时间的时候使用（如：`sleep`）：
```
func entersyscallblock() {
	...... // 与上面函数类似
	// 关键在于此函数
	systemstack(entersyscallblock_handoff)
	// entersyscallblock_handoff 主要调用了 handoffp(releasep())
}
```
`entersyscallblock_handoff` 函数中调用 `releasep` 将M与P彻底解绑（不存在oldp），并调用 `handoffp` 移交P。

#### 4.3.2 P的状态检查
监控线程 `sysmon` 中会调用以下函数来监控所有 `P` :
```
func retake(now int64) uint32 {
	n := 0
	lock(&allpLock)
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]
		if _p_ == nil {
			continue
		}
		pd := &_p_.sysmontick
		s := _p_.status
		sysretake := false
		if s == _Prunning || s == _Psyscall {
			// 首先记录P的调度次数到P的监控次数中
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
				// 若P的监控次数与调度次数一致，并且过去了10ms，说明P已经10ms没有进行调度，一直执行了，抢占
				preemptone(_p_)
				sysretake = true
			}
		}
		if s == _Psyscall {
			// 如果是系统调用，同上，记录P的调度次数到P的监控次数中
			t := int64(_p_.syscalltick)
			if !sysretake && int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}
			// 如果没有工作要做了，并且有其它M在自旋或空闲的P，并且在10ms以内
			if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			unlock(&allpLock)
			// 防止死锁检测
			incidlelocked(-1)
			// 修改P的状态
			if atomic.Cas(&_p_.status, s, _Pidle) {
				n++
				_p_.syscalltick++
				// 移交P
				handoffp(_p_)
			}
			// 恢复
			incidlelocked(1)
			lock(&allpLock)
		}
	}
	unlock(&allpLock)
	return uint32(n)
}
```

#### 4.3.3 P的移交
当系统调用进入阻塞，那么根据可运行队列的情况、网络轮询情况和线程休眠情况来判断这个 `P` 进入休息还是新建 `M` 继续执行任务
```
func handoffp(_p_ *p) {
	// P中还有协程等待执行，startm新建一个m与p绑定继续调度执行
	if !runqempty(_p_) || sched.runqsize != 0 {
		startm(_p_, false)
		return
	}

	// 如果没有工作要做，并且此时没有自旋M，启动一个M自旋找工作
	if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 && atomic.Cas(&sched.nmspinning, 0, 1) { // TODO: fast atomic
		startm(_p_, true)
		return
	}
	lock(&sched.lock)
	// 全局队列中有等待运行的G，新建M绑定这个P
	if sched.runqsize != 0 {
		unlock(&sched.lock)
		startm(_p_, false)
		return
	}
	// 其它P都在休息，并且没人网络轮询，新建M
	if sched.npidle == uint32(gomaxprocs-1) && atomic.Load64(&sched.lastpoll) != 0 {
		unlock(&sched.lock)
		startm(_p_, false)
		return
	}

	when := nobarrierWakeTime(_p_)
	// P放入空闲队列
	pidleput(_p_)
	unlock(&sched.lock)
	// 如果需要网络轮询，唤醒
	if when != 0 {
		wakeNetPoller(when)
	}
}
```

#### 4.3.4 退出系统调用
结束系统调用会调用以下函数：
```
func exitsyscall() {
	_g_ := getg()

	_g_.m.locks++ // see comment in entersyscall
	if getcallersp() > _g_.syscallsp {
		throw("exitsyscall: syscall frame is no longer valid")
	}

	_g_.waitsince = 0
	oldp := _g_.m.oldp.ptr()
	_g_.m.oldp = 0
	// 快速路径 这里是M重新与oldp绑定或者成功获取一个空闲的P
	if exitsyscallfast(oldp) {
		_g_.m.p.ptr().syscalltick++
		// 修改状态
		casgstatus(_g_, _Gsyscall, _Grunning)

		if sched.disable.user && !schedEnabled(_g_) {
			// 主动让出cpu
			Gosched()
		}
		return
	}

	_g_.sysexitticks = 0

	_g_.m.locks--

	// 慢速路径，
	mcall(exitsyscall0)
	_g_.syscallsp = 0
	_g_.m.p.ptr().syscalltick++
	_g_.throwsplit = false
}
// 慢速路径退出系统调用
func exitsyscall0(gp *g) {
	_g_ := getg()
	// 修改G状态为可运行
	casgstatus(gp, _Gsyscall, _Grunnable)
	// 与M解绑
	dropg()
	lock(&sched.lock)
	var _p_ *p
	if schedEnabled(_g_) {
		// 再次尝试获取P
		_p_ = pidleget()
	}
	if _p_ == nil {
		// 没有P 放入全局队列
		globrunqput(gp)
	} else if atomic.Load(&sched.sysmonwait) != 0 {
		atomic.Store(&sched.sysmonwait, 0)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
	if _p_ != nil {
		// 绑定P 继续执行
		acquirep(_p_)
		// 这里永不返回，调用execute运行之后，无论是被抢占(走信号抢占调度)还是执行完毕(走goexit0调度)都不会返回
		execute(gp, false) // Never returns.
	}
	if _g_.m.lockedg != 0 {
		// 有锁的g，等待运行
		stoplockedm()
		// 同上
		execute(gp, false) // Never returns.
	}
	// 这里是没找到P 停止M
	stopm()
	// 上面停止M 被唤醒说明有工作了，这里继续调度
	schedule() // Never returns.
}
```

### 4.4 抢占调度
刚开始的 `go` 的抢占调度是协作式调度，1.14版本后的抢占调度加入了基于信号的抢占。在 `retake` 函数中，如果满足抢占条件，会调用下面的抢占函数：
```
func preemptone(_p_ *p) bool {
	mp := _p_.m.ptr()
	if mp == nil || mp == getg().m {
		return false
	}
	gp := mp.curg
	if gp == nil || gp == mp.g0 {
		return false
	}

	gp.preempt = true
	gp.stackguard0 = stackPreempt

	// 发送抢占信号给需要抢占的线程
	if preemptMSupported && debug.asyncpreemptoff == 0 {
		_p_.preempt = true
		preemptM(mp)
	}

	return true
}
```
`preemptM` 发送抢占信号，内部是调用操作系统的 `tgkill` 发送抢占信号到对应的线程。

#### 4.4.1 基于协作的抢占调度
协作抢占的核心在于编译器会对函数调用前插入 `morestack`，函数调用时会在 `newstack` 中检查是否被抢占，如果被抢占则主动让出cpu。这个过程完全是基于被强占的线程主动发现自己被强占，主动去让出cpu，而且必须基于函数调用才有可能主动发现。
```
func newstack() {
	thisg := getg()
	gp := thisg.m.curg

	// 获取抢占状态
	preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

	if preempt {
		// 无法被强占
		if !canPreemptM(thisg.m) {
			// 继续运行
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	if preempt {
		if gp == thisg.m.g0 {
			// g0 不能被抢占
			throw("runtime: preempt g0")
		}
		if thisg.m.p == 0 && thisg.m.locks == 0 {
			throw("runtime: g is running but p is not")
		}

		if gp.preemptShrink {
			// 栈收缩
			gp.preemptShrink = false
			shrinkstack(gp)
		}

		if gp.preemptStop {
			// 抢占函数
			preemptPark(gp) // never returns
		}

		// 抢占函数
		gopreempt_m(gp) // never return
	}
}
```
`gopreempt_m` 函数调用的就是 4.2 介绍的 `goschedImpl` 函数来将当前 `g` 放入全局可运行队列，并重新进行调度。
#### 4.4.2 基于信号的抢占调度
信号调度的核心在于信号的处理，程序启动时会调用 `initsig` 注册所有信号，对每种信号调用 `setsig` 进行注册处理函数为汇编函数 `sigtramp` ：
```
func setsig(i uint32, fn uintptr) {
	var sa sigactiont
	sa.sa_flags = _SA_SIGINFO | _SA_ONSTACK | _SA_RESTORER | _SA_RESTART
	sigfillset(&sa.sa_mask)
	if GOARCH == "386" || GOARCH == "amd64" {
		sa.sa_restorer = funcPC(sigreturn)
	}
	if fn == funcPC(sighandler) {
		if iscgo {
			fn = funcPC(cgoSigtramp)
		} else {
			// 非cgo的信号处理函数
			fn = funcPC(sigtramp)
		}
	}
	sa.sa_handler = fn
	// 系统调用注册信号
	sigaction(i, &sa, nil)
}
```
最终信号处理函数依然是 `sighandler` 来处理信号。
```
func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
	_g_ := getg()
	c := &sigctxt{info, ctxt}
	if sig == sigPreempt && debug.asyncpreemptoff == 0 {
		// 处理抢占信号
		doSigPreempt(gp, c)
	}
}

func doSigPreempt(gp *g, ctxt *sigctxt) {
	// (gp.preempt || gp.m.p != 0 && gp.m.p.ptr().preempt) && readgstatus(gp)&^_Gscan == _Grunning
	// 检查g的信号状态，g的运行状态
	if wantAsyncPreempt(gp) {
		if ok, newpc := isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()); ok {
			// 修改运行上下文，使用asyncPreempt处理信号
			ctxt.pushCall(funcPC(asyncPreempt), newpc)
		}
	}
	// 增加信号处理次数
	atomic.Xadd(&gp.m.preemptGen, 1)
	// 修改m为没有信号
	atomic.Store(&gp.m.signalPending, 0)

	if GOOS == "darwin" || GOOS == "ios" {
		atomic.Xadd(&pendingPreemptSignals, -1)
	}
}
```
`asyncPreempt` 函数调用以下函数来处理抢占信号：
```
func asyncPreempt2() {
	gp := getg()
	gp.asyncSafePoint = true
	if gp.preemptStop {
		mcall(preemptPark)
	} else {
		mcall(gopreempt_m)
	}
	gp.asyncSafePoint = false
}
```
可以看到抢占信号的处理与 `newstack` 中的对抢占的处理是一样的。
```
func preemptPark(gp *g) {
	if trace.enabled {
		traceGoPark(traceEvGoBlock, 0)
	}
	status := readgstatus(gp)
	if status&^_Gscan != _Grunning {
		dumpgstatus(gp)
		throw("bad g status")
	}
	gp.waitreason = waitReasonPreempted
	// 这里最终G的状态为_Gpreempted。
	casGToPreemptScan(gp, _Grunning, _Gscan|_Gpreempted)
	dropg()
	casfrom_Gscanstatus(gp, _Gscan|_Gpreempted, _Gpreempted)
	// 调度
	schedule()
}
```
这里解释一下 `preemptStop` ，信号抢占调度提供了 `suspendG` 函数来挂起一个线程，此函数会 `preemptStop = true` ，如果是被其它线程挂起了被抢占的线程，那么被强占线程中的 `G` 的状态为 `_Gpreempted`。而调用 `suspendG` 的线程会在此函数中进行状态管理：
```
func suspendG(gp *g) suspendGState {
 if mp := getg().m; mp.curg != nil && readgstatus(mp.curg) == _Grunning {
	 throw("suspendG from non-preemptible goroutine")
 }

 const yieldDelay = 10 * 1000
 var nextYield int64

 stopped := false
 for i := 0; ; i++ {
	 switch s := readgstatus(gp); s {
	 default:

	 case _Gpreempted:
		 // 转换为 _Gwaiting
		 if !casGFromPreempted(gp, _Gpreempted, _Gwaiting) {
			 break
		 }

		 stopped = true

		 s = _Gwaiting
		 // 继续运行
		 fallthrough

	 case _Grunnable, _Gsyscall, _Gwaiting:
		 if !castogscanstatus(gp, s, s|_Gscan) {
			 break
		 }

		 // 清楚调度状态
		 gp.preemptStop = false
		 gp.preempt = false
		 gp.stackguard0 = gp.stack.lo + _StackGuard

		 // 返回被挂起的协程
		 return suspendGState{g: gp, stopped: stopped}
	 }
 }
}
```
这样发起抢占的线程就拥有了被抢占的协程的所有权，可以自主调用 `resumeG` 恢复执行（最终通过 `ready` 来恢复执行协程）。那么这两种抢占调度是否会冲突？并不会，因为无论是协作调度还是信号调度都会是被抢占的协程来执行，一定有一个先后顺序，如果其中一个调度成功，另一个检查是否需要调度的时候就不会通过。最终被抢占的协程再次被调度到执行时，会重置它所有的抢占状态。

## 总结
协程调度实现非常复杂，而且涉及了内存管理，网络轮询，定时器等等逻辑。这里忽略了涉及的其它模块，仅仅解释了调度的种类和过程。
