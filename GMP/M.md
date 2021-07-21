# M
`M` 表示一个系统线程

## 1. M的主要字段
```
type m struct {
	g0      *g     // 调用栈
	morebuf gobuf  // gobuf arg to morestack

	gsignal       *g           // 处理信号用的协程
	tls           [6]uintptr   // thread-local storage (for x86 extern register)
	mstartfn      func()   // 开始执行的函数
	curg          *g       // 当前正在执行的G
	p             puintptr // 关联的P
	nextp         puintptr // 即将使用的p
	oldp          puintptr // the p that was attached before executing a syscall
	id            int64
	mallocing     int32

	preemptoff    string // if != "", keep curg running on this m
	locks         int32  // 锁 禁止抢占

	spinning      bool // m 自旋
	blocked       bool // m 阻塞
	newSigstack   bool // minit on C thread called sigaltstack

	freeWait      uint32 // 如果等于0，可以安全的释放g0并且删除m
	fastrand      [2]uint32 // 随机用的
	park          note // M的挂起锁
	alllink       *m // on allm
	lockedg       guintptr	// 锁定的G
	waitunlockf   func(*g, unsafe.Pointer) bool	// 挂起当前的G要执行的函数 可以为nil
	waitlock      unsafe.Pointer  // 上面函数的第二个参数

	freelink      *m // on sched.freem 链表

	// M成功处理信号的次数
	preemptGen uint32

	// 这个M是否有信号没处理
	signalPending uint32
}
```

## 2. M的创建
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
### 2.1 M的分配
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
## 3. M的运行时

### 3.1 启动一个M
启动 `M` 使用以下函数：
```
func startm(_p_ *p, spinning bool) {
	mp := acquirem()
	lock(&sched.lock)
	if _p_ == nil {
		// 获取一个空闲的P
		_p_ = pidleget()
		if _p_ == nil {
			unlock(&sched.lock)
			// 获取失败
			if spinning {
				if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
					throw("startm: negative nmspinning")
				}
			}
			// 不启动m了 没有p
			releasem(mp)
			return
		}
	}
	// 获取一个空闲的M
	nmp := mget()
	if nmp == nil {
		// 没有空闲的M 新建一个
		id := mReserveID()
		unlock(&sched.lock)

		var fn func()
		if spinning {
			fn = mspinning
		}
		newm(fn, _p_, id)
		releasem(mp)
		return
	}
	unlock(&sched.lock)
	if nmp.spinning {
		throw("startm: m is spinning")
	}
	if nmp.nextp != 0 {
		throw("startm: m has p")
	}
	if spinning && !runqempty(_p_) {
		throw("startm: p has runnable gs")
	}
	nmp.spinning = spinning
	// 给被唤醒的M 准备好P
	nmp.nextp.set(_p_)
	// 唤醒M
	notewakeup(&nmp.park)
	releasem(mp)
}
```
有空闲的 `M` 就是用，没有的话新建。但是必须有相应的 `P` 进行绑定。**需要注意的是这是在一个线程中启动另一个线程**

### 3.2 停止M
```
func stopm() {
	_g_ := getg()
	// m被加锁了 还在运行相关东西
	if _g_.m.locks != 0 {
		throw("stopm holding locks")
	}
	// m还持有p
	if _g_.m.p != 0 {
		throw("stopm holding p")
	}
	// m是自旋的
	if _g_.m.spinning {
		throw("stopm spinning")
	}

	lock(&sched.lock)
	// 放入空闲队列
	mput(_g_.m)
	unlock(&sched.lock)
	// m休眠
	mPark()
	// m被唤醒了
	acquirep(_g_.m.nextp.ptr())
	_g_.m.nextp = 0
}
```
**与上面对应，空闲的M被唤醒后继续执行，被唤醒前P已经被准备好了**

### 3.3 M的休眠
```
func mPark() {
	g := getg()
	for {
		notesleep(&g.m.park)
		noteclear(&g.m.park)
		if !mDoFixup() {
			return
		}
	}
}
```
`mPark` 是唯一使 `M` 挂起的函数，它挂起的是一个真正的系统线程（要与 `suspendG` 协程挂起区分开 ）。  
**结合调度文档中，可以看到一个M当没有P可与之绑定时（执行阻塞系统调用结束）会被休眠放入全局M空闲队列等待唤醒使用。这样做可以不频繁创建系统线程，其实本质是个线程池**

## 4. M的消亡
`M` 可能存在的几种状态：
1. 不绑定 `P`：
	1. 运行监控线程、模版线程等
	2. 正在执行阻塞系统调用
	3. 休眠状态
2. 绑定 `P`：
	1. 正在调度过程中
	2. 执行用户代码过程中
	3. 自旋寻找工作过程中  

从上面的这些状态来看，`M` 一般不会退出，要不休眠要不工作。  
**似乎除了出错或特殊情况（LockOSThread），那么不负责任的猜测一下：是否某一个时刻阻塞系统调用超级多的时候，我们会有大量的系统线程被创建达到一个峰值（毕竟会复用），然后这些线程一直会是idle状态？是否可以利用LockOSThread来解决大量的线程？**  
`mstart` 函数最后一行代码为 `mexit`。这个函数就是退出线程，主要调用了系统函数 `syscall $sys_exit`。  
那么再次回顾 `mstart1` ，一进去就保存了 `sp` 和 `pc` ，因此注释不推荐直接使用 `mexit`，取而代之使用 `gogo(&_g_.m.g0.sched)` 来退出线程（使用 `gogo` 来对栈进行退回到 `mexit` 这个函数前）。

## 一点小补充 LockOSThread
`LockOSThread` 可以将 `M` 和 `G` 进行绑定，使 `M` 只能运行绑定的 `G` （其实是在调度前，查看是否有绑定的 `G`，有则运行即可）
```
func dolockOSThread() {
	_g_ := getg()
	_g_.m.lockedg.set(_g_)
	_g_.lockedm.set(_g_.m)
}
```
**一般是调用c库使用，也可以通过此手段让我们某个重要的协程一直运行**  
**如果调用了LockOSThread，知道协程执行结束，依然没有调用UnlockOSThread，那么我们协程绑定的线程，就会直接退出！**

## 两点小补充 notewakeup和notesleep
这两个函数是系统线程 `M` 休眠和唤醒的核心函数。内部主要封装和使用了系统调用 `futex` 来进行线程的休眠和唤醒
