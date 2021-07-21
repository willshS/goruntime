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
    rseq    uintptr   // 此结构体重用或定时器被重置
    rg      uintptr   // 状态
    rt      timer     // 读定时器
    rd      int64     // 定时器时间
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
此函数核心在于对于有事件触发时对 `goroutine` 的处理。
```
// 此函数是挂起读或写的goroutine，并将pd.rg 修改为 当前goroutine
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

    // 挂起协程 netpollblockcommit 函数中将pd.rg = goroutine
    if waitio || netpollcheckerr(pd, mode) == 0 {
        gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
    }
    // 被唤醒 修改pd.rg
    old := atomic.Xchguintptr(gpp, 0)
    if old > pdWait {
        throw("runtime: corrupted polldesc")
    }
    return old == pdReady
}

// 此函数取出上面挂起的goroutine 并返回出去等待调度
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
    gpp := &pd.rg
    if mode == 'w' {
        gpp = &pd.wg
    }

    for {
        old := *gpp
        if old == pdReady {
            return nil
        }
        if old == 0 && !ioready {
            return nil
        }
        var new uintptr
        if ioready {
            new = pdReady
        }
        // 修改状态 如果是epoll_wait 这里就被修改为ready 对应上面挂起协程的唤醒检查 否则修改为0
        if atomic.Casuintptr(gpp, old, new) {
            if old == pdWait {
                old = 0
            }
            return (*g)(unsafe.Pointer(old))
        }
    }
}
```
网络轮询器是线程安全的。上面是最核心的代码，将包含 `goroutine` 指针的 `pollDesc` 注册到 `epoll` 与 `redis` 的将事件数据结构（包含读写回调函数）注册到 `epoll` 其实是一个道理。当要读取数据或写入数据时，尝试直接读取或写入，若失败则将当前 `goroutine` 挂起，等待网络轮询触发事件将挂起的协程返回给外部等待调度。  
**这里要提一下网络轮询的调用在监控线程实现中的 `sysmon` 和 调度算法中的 `findrunnable` 中进行。最开始的全局管道就是因为 `timer` 这个东西是在 `P` 上的，所以在阻塞等待网络轮询的时候可能需要打断检查一下 `timer` 。**

## 5. deadline
### 5.1 设置deadline
```
func poll_runtime_pollSetDeadline(pd *pollDesc, d int64, mode int) {
    lock(&pd.lock)
    if pd.closing {
        unlock(&pd.lock)
        return
    }
    rd0, wd0 := pd.rd, pd.wd
    combo0 := rd0 > 0 && rd0 == wd0
    // 计算d
    if d > 0 {
        d += nanotime()
        if d <= 0 {
            // 溢出处理
            d = 1<<63 - 1
        }
    }
    if mode == 'r' || mode == 'r'+'w' {
        pd.rd = d
    }
    if mode == 'w' || mode == 'r'+'w' {
        pd.wd = d
    }
    combo := pd.rd > 0 && pd.rd == pd.wd
    rtf := netpollReadDeadline
    if combo {
        rtf = netpollDeadline
    }
    if pd.rt.f == nil {
        if pd.rd > 0 {
            // 设置读超时函数为netpollReadDeadline
            pd.rt.f = rtf
            // 将当前pd和rseq赋值为timer参数，
            pd.rt.arg = pd.makeArg()
            pd.rt.seq = pd.rseq
            // 设置timer
            resettimer(&pd.rt, pd.rd)
        }
    } else if pd.rd != rd0 || combo != combo0 {
        pd.rseq++ // 如果当前已经设置了r deadline，这里进行修改
        if pd.rd > 0 {
            // 修改timer
            modtimer(&pd.rt, pd.rd, 0, rtf, pd.makeArg(), pd.rseq)
        } else {
            // 删除timer
            deltimer(&pd.rt)
            pd.rt.f = nil
        }
    }
    // 省略写
    // 如果d小于0，唤醒挂起的协程，但是pd.rg或pd.wg状态是0，也就是没有数据
    var rg, wg *g
    if pd.rd < 0 || pd.wd < 0 {
        atomic.StorepNoWB(noescape(unsafe.Pointer(&wg)), nil)
        if pd.rd < 0 {
            // 取出协程
            rg = netpollunblock(pd, 'r', false)
        }
        if pd.wd < 0 {
            wg = netpollunblock(pd, 'w', false)
        }
    }
    unlock(&pd.lock)
    if rg != nil {
        // 唤醒
        netpollgoready(rg, 3)
    }
    if wg != nil {
        netpollgoready(wg, 3)
    }
}
```
### 5.2 timer到期函数
```
func netpolldeadlineimpl(pd *pollDesc, seq uintptr, read, write bool) {
    lock(&pd.lock)
    currentSeq := pd.rseq
    if !read {
        currentSeq = pd.wseq
    }
    if seq != currentSeq {
        // seq变了，说明timer重置了
        unlock(&pd.lock)
        return
    }
    var rg *g
    if read {
        if pd.rd <= 0 || pd.rt.f == nil {
            throw("runtime: inconsistent read deadline")
        }
        // 这里把rd赋值为-1
        pd.rd = -1
        atomic.StorepNoWB(unsafe.Pointer(&pd.rt.f), nil) // full memory barrier between store to rd and load of rg in netpollunblock
        // 取出挂起的协程
        rg = netpollunblock(pd, 'r', false)
    }
    unlock(&pd.lock)
    if rg != nil {
        // 唤醒
        netpollgoready(rg, 0)
    }
}
```
**这里要与第4节的状态一起看。状态有pdready、pdwait、0、G这几种。**
**如果协程进来等待，则状态被置为pdwait，然后被修改为挂起的G**
1. **有数据唤醒，即调用epoll_wait被唤醒，状态被置为pdready，并调度挂起的G**
2. **若timer到期被唤醒，则状态被置为0，并调度挂起的G**
3. **被唤醒的G检查唤醒时的状态，若为0则代表没有数据此时rd为-1，返回调用者timeout。若为pdready，则表示有数据，去读写即可。最后将状态再次修改为0**  

以上是网络轮询器对于状态的转换，是实现 `deadline` 的核心。

## 6. 总结
网络轮询器的重点有两个：
1. 注册 `pd` 到 `epoll`。挂起等待数据的协程并赋值给 `pd`
2. 数据就绪或时间到期后从 `epoll` 中读取 `pd` 的协程并唤醒
3. 被唤醒的协程检查状态，判断是数据就绪就去读取，判断是时间到期则返回 `timeout`
