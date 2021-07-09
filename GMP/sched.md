# sched
关于GMP，网上特别多博客资料。  
[写得非常好的总结](https://github.com/fuweid/notes/blob/master/golang/goroutine_scheduler_overview.md)  
[官方的一些说明](https://github.com/golang/go/blob/048c9cfaacb6fe7ac342b0acd8ca8322b6c49508/src/runtime/HACKING.md)  
[官方的设计文档](https://golang.org/s/go11sched)  
[draveness大佬写的书](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)

## 调度的全局参数
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

## 起始
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

## 初始化
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

## main函数的开始
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

	// Make racy client program work: if panicking on
	// another goroutine at the same time as main returns,
	// let the other goroutine finish printing the panic trace.
	// Once it does, it will exit. See issues 3934 and 20018.
	if atomic.Load(&runningPanicDefers) != 0 {
		// Running deferred functions should not take long.
		for c := 0; c < 1000; c++ {
			if atomic.Load(&runningPanicDefers) == 0 {
				break
			}
			Gosched()
		}
	}
	if atomic.Load(&panicking) != 0 {
		gopark(nil, nil, waitReasonPanicWait, traceEvGoStop, 1)
	}

	exit(0)
	for {
		var x *int32
		*x = 0
	}
}
```
