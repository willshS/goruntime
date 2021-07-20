# 网络轮询器
`go` 的网络模块实现其实与 `redis` 或 `libuv` 等服务或网络库从根本上来说没什么不同，封装不同系统的 `api` 。

## 1. 轮询器
以下是网络轮询器的核心数据结构。
```
type pollDesc struct {
	link *pollDesc // 缓存池指针
	lock    mutex // 锁
	fd      uintptr // 系统fd
	closing bool
	everr   bool      // error发生
	user    uint32    // 部分系统使用？
	rseq    uintptr   // protects from stale read timers
	rg      uintptr   // pdReady, pdWait, G waiting for read or nil
	rt      timer     // 读定时器
	rd      int64     // read deadline
	wseq    uintptr   // protects from stale write timers
	wg      uintptr   // pdReady, pdWait, G waiting for write or nil
	wt      timer     // write deadline timer
	wd      int64     // write deadline
	self    *pollDesc // 存储自身，为转换接口使用
}
```

## 2. 初始化
打开文件或创建 `socket` 都会对网络轮询进行初始化：
```
func poll_runtime_pollServerInit() {
	netpollGenericInit()
}

func netpollGenericInit() {
  // 初始化标志 只会初始化一次
	if atomic.Load(&netpollInited) == 0 {
		lockInit(&netpollInitLock, lockRankNetpollInit)
		lock(&netpollInitLock)
    // 双检查
		if netpollInited == 0 {
      // 调用对应系统的初始化 epoll kqueue windows的io完成端口等
			netpollinit()
			atomic.Store(&netpollInited, 1)
		}
		unlock(&netpollInitLock)
	}
}
```
这里我们只关注 `epoll` 相关实现：
```
func netpollinit() {
  // 创建epoll
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	if epfd < 0 {
		epfd = epollcreate(1024)
		if epfd < 0 {
			println("runtime: epollcreate failed with", -epfd)
			throw("runtime: netpollinit failed")
		}
		closeonexec(epfd)
	}
  // 创建非阻塞管道
	r, w, errno := nonblockingPipe()
	if errno != 0 {
		println("runtime: pipe failed with", -errno)
		throw("runtime: pipe failed")
	}
	ev := epollevent{
		events: _EPOLLIN,
	}
	*(**uintptr)(unsafe.Pointer(&ev.data)) = &netpollBreakRd
  // 注册管道到epoll
	errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)
	if errno != 0 {
		println("runtime: epollctl failed with", -errno)
		throw("runtime: epollctl failed")
	}
  // 管道赋值给全局变量
	netpollBreakRd = uintptr(r)
	netpollBreakWr = uintptr(w)
}
```
这里需要注意的是创建 `epoll` 后，会创建一个管道，并将此管道注册入 `epoll`。小技巧：方便随时打断 `epoll_wait`。

## 3. 添加事件与删除事件
想要监听 `IO` 就要添加相应的事件到 `epoll`。
```
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
  // 缓存分配pollDesc
	pd := pollcache.alloc()
	lock(&pd.lock)
	if pd.wg != 0 && pd.wg != pdReady {
		throw("runtime: blocked write on free polldesc")
	}
	if pd.rg != 0 && pd.rg != pdReady {
		throw("runtime: blocked read on free polldesc")
	}
	pd.fd = fd
	pd.closing = false
	pd.everr = false
	pd.rseq++
	pd.rg = 0
	pd.rd = 0
	pd.wseq++
	pd.wg = 0
	pd.wd = 0
	pd.self = pd
	unlock(&pd.lock)

	var errno int32
  // 添加事件
	errno = netpollopen(fd, pd)
	return pd, int(errno)
}

func netpollopen(fd uintptr, pd *pollDesc) int32 {
	var ev epollevent
  // 这里读写事件都添加了，并且是ET模式
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
  // 数据直接就是 pollDesc
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```
删除事件与添加相似，仅仅是反操作。代码略过。

## 4. 轮询
轮询函数其实就是 `epoll` 的封装。
```
func netpoll(delay int64) gList {
	if epfd == -1 {
		return gList{}
	}
  // 处理参数
	var waitms int32
	if delay < 0 {
    // 阻塞直到有数据
		waitms = -1
	} else if delay == 0 {
    // 非阻塞轮询一次
		waitms = 0
  // 阻塞一定时间
	} else if delay < 1e6 {
    // 太小了
		waitms = 1
	} else if delay < 1e15 {
		waitms = int32(delay / 1e6)
	} else {
    // 太大了
		// 1e9 ms == ~11.5 days.
		waitms = 1e9
	}
	var events [128]epollevent
retry:
  // 多路io复用
	n := epollwait(epfd, &events[0], int32(len(events)), waitms)
	if n < 0 {
		if n != -_EINTR {
			println("runtime: epollwait on fd", epfd, "failed with", -n)
			throw("runtime: netpoll failed")
		}
		// 让外面自己判断是否还需要epollwait
		if waitms > 0 {
			return gList{}
		}
		goto retry
	}
	var toRun gList
	for i := int32(0); i < n; i++ {
		ev := &events[i]
		if ev.events == 0 {
			continue
		}

		if *(**uintptr)(unsafe.Pointer(&ev.data)) == &netpollBreakRd {
			if ev.events != _EPOLLIN {
				println("runtime: netpoll: break fd ready for", ev.events)
				throw("runtime: netpoll: break fd ready for something unexpected")
			}
			if delay != 0 {
				// 用全局通道打断
				var tmp [16]byte
				read(int32(netpollBreakRd), noescape(unsafe.Pointer(&tmp[0])), int32(len(tmp)))
				atomic.Store(&netpollWakeSig, 0)
			}
			continue
		}

		var mode int32
		if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'r'
		}
		if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'w'
		}
		if mode != 0 {
			pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
			pd.everr = false
			if ev.events == _EPOLLERR {
				pd.everr = true
			}
      // 取出goroutine
			netpollready(&toRun, pd, mode)
		}
	}
	return toRun
}
```
此函数核心在于对于有事件触发时对 `goroutine` 的处理。先来看当我们读取或写入数据时没有数据或缓冲区的情况：
```
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}
  // 修改pd中的状态为pdWait
	for {
		old := *gpp
		if old == pdReady {
      // 数据可能被其他G刚刚取走
			*gpp = 0
			return true
		}
		if old != 0 {
			throw("runtime: double wait")
		}
		if atomic.Casuintptr(gpp, 0, pdWait) {
			break
		}
	}

	// need to recheck error states after setting gpp to pdWait
	// this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
	// do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
	if waitio || netpollcheckerr(pd, mode) == 0 {
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}
	// be careful to not lose concurrent pdReady notification
	old := atomic.Xchguintptr(gpp, 0)
	if old > pdWait {
		throw("runtime: corrupted polldesc")
	}
	return old == pdReady
}
```
