# timer
定时器 `timer` 是代码中非常重要的一个功能。

## 1. 使用
```
func main() {
    x := time.Now()
    fmt.Println(x)
    // after
    c := time.After(time.Second * 1)
    select {
    case t := <-c:
        fmt.Println(t)
    }
    // after func
    time.AfterFunc(time.Second, func() {
        fmt.Println(time.Now())
    })
    // tick
    c = time.Tick(time.Second * 1)
    select {
    case t := <-c:
        fmt.Println(t)
    }
    // ticker
    ticker := time.NewTicker(time.Second)
    select {
    case t := <-ticker.C:
        fmt.Println(t)
    }
    // timer
    timer := time.NewTimer(time.Second)
    select {
    case t := <-timer.C:
        fmt.Println(t)
    }
}
```
其实无论是 `tick` 还是 `timer` 又或者 `sleep`，内部都是使用 `runtime` 包的定时器实现。

## 2. runtime.timer
```
type timer struct {
    pp puintptr // timer所在的P
    when   int64 // timer的到期时间
    period int64 // 周期 第一次触发后每隔period再次触发
    f      func(interface{}, uintptr) // timer到期执行的函数
    arg    interface{} // f的参数
    seq    uintptr // 主要是netpoll在用 f的第二个参数
    nextwhen int64 // timer被修改后的时间
    status uint32 // timer的状态
}
```
### 2.1 addtimer
`addtimer` 是所有定时器创建的核心函数：
```
// addtimer 状态变化:
//   timerNoStatus   -> timerWaiting
//   anything else   -> panic: invalid value
func addtimer(t *timer) {
    // 安全检查
    // 修改timer状态为waiting等待执行
    t.status = timerWaiting
    when := t.when
    // 获取当前G的P
    pp := getg().m.p.ptr()
    lock(&pp.timersLock)
    // 清洗一下P上的timer
    cleantimers(pp)
    // 加入到P的timer切片中，并调整切片顺序（本质上是个四叉堆）
    doaddtimer(pp, t)
    unlock(&pp.timersLock)
    // 如果需要的话唤醒netpoll
    wakeNetPoller(when)
}
```
**这里唤醒netpoll是为了更快调度到距离当前时间比较近的timer。比如有一个P正在阻塞netpoll，过期时间为5秒，此时有一个协程创建了一个timer过期时间为1秒，那么此时打断能保证新的timer可以更快被调度到。**
```
func doaddtimer(pp *p, t *timer) {
    // timer依赖于netpoll
    if netpollInited == 0 {
        netpollGenericInit()
    }

    if t.pp != 0 {
        throw("doaddtimer: P already set in timer")
    }
    t.pp.set(pp)
    i := len(pp.timers)
    // 增加到本地P的timer队列
    pp.timers = append(pp.timers, t)
    // 调整四叉堆
    siftupTimer(pp.timers, i)
    if t == pp.timers[0] {
        // 如果是最小的，修改P的最近timer时间
        atomic.Store64(&pp.timer0When, uint64(t.when))
    }
    atomic.Xadd(&pp.numTimers, 1)
}
```
### 2.2 deltimer
因为偷窃调度的存在，删除操作并不是真的删除，而是对定时器做一个标记。
```
// deltimer 状态机:
//   timerWaiting         -> timerModifying -> timerDeleted
//   timerModifiedEarlier -> timerModifying -> timerDeleted
//   timerModifiedLater   -> timerModifying -> timerDeleted
//   timerNoStatus        -> do nothing
//   timerDeleted         -> do nothing
//   timerRemoving        -> do nothing
//   timerRemoved         -> do nothing
//   timerRunning         -> wait until status changes
//   timerMoving          -> wait until status changes
//   timerModifying       -> wait until status changes
func deltimer(t *timer) bool {
    for {
        switch s := atomic.Load(&t.status); s {
        case timerWaiting, timerModifiedLater:
            mp := acquirem()
            if atomic.Cas(&t.status, s, timerModifying) {
                // 因为偷窃调度，timer可能在其它P上，只能标记为删除
                tpp := t.pp.ptr()
                if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
                    badTimer()
                }
                releasem(mp)
                atomic.Xadd(&tpp.deletedTimers, 1)
                return true
            } else {
                releasem(mp)
            }
        case timerModifiedEarlier:
            mp := acquirem()
            if atomic.Cas(&t.status, s, timerModifying) {
                tpp := t.pp.ptr()
                // 与上面不同的是多一个减法
                atomic.Xadd(&tpp.adjustTimers, -1)
                if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
                    badTimer()
                }
                releasem(mp)
                atomic.Xadd(&tpp.deletedTimers, 1)
                // Timer was not yet run.
                return true
            } else {
                releasem(mp)
            }
        case timerDeleted, timerRemoving, timerRemoved:
            // timer已经运行过了
            return false
        case timerRunning, timerMoving:
            // timer正在其它P运行或移动
            osyield()
        case timerNoStatus:
            return false
        case timerModifying:
            // 正在被修改
            osyield()
        default:
            badTimer()
        }
    }
}
```

### 2.3 modtimer
修改定时器有两种，一种是提前，一种是延后。使用 `nextwhen` 字段来存储新的 `when`。
```
// modtimer 状态机:
//   timerWaiting    -> timerModifying -> timerModifiedXX
//   timerModifiedXX -> timerModifying -> timerModifiedYY
//   timerNoStatus   -> timerModifying -> timerWaiting
//   timerRemoved    -> timerModifying -> timerWaiting
//   timerDeleted    -> timerModifying -> timerModifiedXX
//   timerRunning    -> wait until status changes
//   timerMoving     -> wait until status changes
//   timerRemoving   -> wait until status changes
//   timerModifying  -> wait until status changes
func modtimer(t *timer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr) bool {
    status := uint32(timerNoStatus)
    wasRemoved := false
    var pending bool
    var mp *m
loop:
    for {
        switch status = atomic.Load(&t.status); status {
        case timerWaiting, timerModifiedEarlier, timerModifiedLater:
            mp = acquirem()
            if atomic.Cas(&t.status, status, timerModifying) {
                pending = true // timer 还没有运行
                break loop
            }
            releasem(mp)
        case timerNoStatus, timerRemoved:
            mp = acquirem()
            // timer已经运行结束并且已经被从P的堆上移除了
            if atomic.Cas(&t.status, status, timerModifying) {
                wasRemoved = true
                pending = false // timer already run or stopped
                break loop
            }
            releasem(mp)
        case timerDeleted:
            // timer删除了 此时还在堆上
            mp = acquirem()
            if atomic.Cas(&t.status, status, timerModifying) {
                atomic.Xadd(&t.pp.ptr().deletedTimers, -1)
                pending = false // timer already stopped
                break loop
            }
            releasem(mp)
        case timerRunning, timerRemoving, timerMoving:
            osyield()
        case timerModifying:
            osyield()
        default:
            badTimer()
        }
    }
    // 更新timer
    t.period = period
    t.f = f
    t.arg = arg
    t.seq = seq

    if wasRemoved {
        // 如果已经被移除了，重新加到P的堆上
        t.when = when
        pp := getg().m.p.ptr()
        lock(&pp.timersLock)
        doaddtimer(pp, t)
        unlock(&pp.timersLock)
        if !atomic.Cas(&t.status, timerModifying, timerWaiting) {
            badTimer()
        }
        releasem(mp)
        wakeNetPoller(when)
    } else {
        // 如果在其它P上，用nextwhen过度
        t.nextwhen = when
        newStatus := uint32(timerModifiedLater)
        if when < t.when {
            newStatus = timerModifiedEarlier
        }
        // 获取所在的P
        tpp := t.pp.ptr()
        adjust := int32(0)
        if status == timerModifiedEarlier {
            // 如果以前的状态是earlier 可以减少一次调整
            adjust--
        }
        if newStatus == timerModifiedEarlier {
            // 新的状态是earlier 增加一次调整
            adjust++
            // 更新P的最小when
            updateTimerModifiedEarliest(tpp, when)
        }
        if adjust != 0 {
            atomic.Xadd(&tpp.adjustTimers, adjust)
        }

        // 设置最新的状态
        if !atomic.Cas(&t.status, timerModifying, newStatus) {
            badTimer()
        }
        releasem(mp)
        // 如果被更改为提前了，唤醒netpoll
        if newStatus == timerModifiedEarlier {
            wakeNetPoller(when)
        }
    }

    return pending
}
```
### 2.4 adjusttimers
以上的修改和删除都是修改状态，真正的修改和删除是通过调用 `adjusttimers` 。此函数的调用是在 `P` 进行调度的时候，此时修改 `P` 上的所有需要修改的定时器
```
func adjusttimers(pp *p, now int64) {
    if atomic.Load(&pp.adjustTimers) == 0 {
        if verifyTimers {
            verifyTimerHeap(pp)
        }
        // 没有需要调整的timer  需要真正删除的可以不用着急
        atomic.Store64(&pp.timerModifiedEarliest, 0)
        return
    }

    // 如果被修改的最早的时间点还没到，也可以不用急
    if first := atomic.Load64(&pp.timerModifiedEarliest); first != 0 {
        if int64(first) > now {
            if verifyTimers {
                verifyTimerHeap(pp)
            }
            return
        }

        // 否则就要修改
        atomic.Store64(&pp.timerModifiedEarliest, 0)
    }

    var moved []*timer
loop:
    for i := 0; i < len(pp.timers); i++ {
        t := pp.timers[i]
        if t.pp.ptr() != pp {
            throw("adjusttimers: bad p")
        }
        switch s := atomic.Load(&t.status); s {
        case timerDeleted:
            if atomic.Cas(&t.status, s, timerRemoving) {
                // 真正的从P的堆上删除，这里也主要是调整四叉堆
                dodeltimer(pp, i)
                if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
                    badTimer()
                }
                atomic.Xadd(&pp.deletedTimers, -1)
                // 再次从堆头开始
                i--
            }
        case timerModifiedEarlier, timerModifiedLater:
            if atomic.Cas(&t.status, s, timerMoving) {
                // Now we can change the when field.
                t.when = t.nextwhen
                // 修改的先删除掉
                dodeltimer(pp, i)
                moved = append(moved, t)
                if s == timerModifiedEarlier {
                    // 如果后面没有需要调整的 跳出循环
                    if n := atomic.Xadd(&pp.adjustTimers, -1); int32(n) <= 0 {
                        break loop
                    }
                }
                // Look at this heap position again.
                i--
            }
        case timerNoStatus, timerRunning, timerRemoving, timerRemoved, timerMoving:
            badTimer()
        case timerWaiting:
            // OK, nothing to do.
        case timerModifying:
            // 等修改完再次检查
            osyield()
            i--
        default:
            badTimer()
        }
    }

    if len(moved) > 0 {
        // 将需要修改的再次添加到堆上，并修改状态为等待
        addAdjustedTimers(pp, moved)
    }

    if verifyTimers {
        verifyTimerHeap(pp)
    }
}
```

## 3. 定时器的触发
定时器的触发是在调度过程中对定时器进行检查，并运行到期的定时器函数：
```
func checkTimers(pp *p, now int64) (rnow, pollUntil int64, ran bool) {
    next := int64(atomic.Load64(&pp.timer0When))
    nextAdj := int64(atomic.Load64(&pp.timerModifiedEarliest))
    if next == 0 || (nextAdj != 0 && nextAdj < next) {
        next = nextAdj
    }

    if next == 0 {
        // 没有定时器需要运行或调整
        return now, 0, false
    }

    if now == 0 {
        now = nanotime()
    }
    if now < next {
        // 最早的定时器时间点还没到，并且需要真正删除的不到1/4 先不管
        if pp != getg().m.p.ptr() || int(atomic.Load(&pp.deletedTimers)) <= int(atomic.Load(&pp.numTimers)/4) {
            return now, next, false
        }
    }
    // 修改定时器堆需要加锁
    lock(&pp.timersLock)

    if len(pp.timers) > 0 {
        // 真正的修改定时器状态
        adjusttimers(pp, now)
        for len(pp.timers) > 0 {
            // 尝试运行定时器
            if tw := runtimer(pp, now); tw != 0 {
                if tw > 0 {
                    // 记录下一个定时器的时间点
                    pollUntil = tw
                }
                // 四叉堆小于now的都运行结束了，不需要继续遍历
                break
            }
            ran = true
        }
    }

    // 删除被标记为删除状态的定时器，并调整堆
    if pp == getg().m.p.ptr() && int(atomic.Load(&pp.deletedTimers)) > len(pp.timers)/4 {
        clearDeletedTimers(pp)
    }
    unlock(&pp.timersLock)
    return now, pollUntil, ran
}
```
`runtimer` 函数会尝试运行四叉堆顶的定时器，它会根据状态先进行调整堆顶的定时器，比如需要删除，需要修改的情况下先进行修改知道碰到一个等待状态的定时器。如果运行成功返回0，如果调整结束定时器为空，返回-1，如果下一个定时器的时间点还没到，返回下一个定时器的时间。
```
func runOneTimer(pp *p, t *timer, now int64) {
    f := t.f
    arg := t.arg
    seq := t.seq

    if t.period > 0 {
        // 如果存在循环触发周期，更新timer的时间点并调整四叉堆
        delta := t.when - now
        t.when += t.period * (1 + -delta/t.period)
        if t.when < 0 { // check for overflow.
            t.when = maxWhen
        }
        siftdownTimer(pp.timers, 0)
        if !atomic.Cas(&t.status, timerRunning, timerWaiting) {
            badTimer()
        }
        updateTimer0When(pp)
    } else {
        // 否则直接删除
        dodeltimer0(pp)
        if !atomic.Cas(&t.status, timerRunning, timerNoStatus) {
            badTimer()
        }
    }
    unlock(&pp.timersLock)
    // 运行时间定时器函数 运行过程中不能加timer的锁，其它P此时可能来偷
    f(arg, seq)
    lock(&pp.timersLock)
}
```

## 4. timer & ticker
`time.NewTimer` 和 `time.NewTicker` 两中定时器的差别其实看一下这两个函数即可：
```
func NewTicker(d Duration) *Ticker {
    if d <= 0 {
        panic(errors.New("non-positive interval for NewTicker"))
    }
    c := make(chan Time, 1)
    t := &Ticker{
        C: c,
        r: runtimeTimer{
            when:   when(d),
            period: int64(d),
            f:      sendTime,
            arg:    c,
        },
    }
    startTimer(&t.r)
    return t
}

func NewTimer(d Duration) *Timer {
    c := make(chan Time, 1)
    t := &Timer{
        C: c,
        r: runtimeTimer{
            when: when(d),
            f:    sendTime,
            arg:  c,
        },
    }
    startTimer(&t.r)
    return t
}
```
可以看到 `ticker` 多了一个安全检查以及 `runtime.timer` 结构体的 `period` 字段。而 `period` 字段在我们执行定时器的时候，会重新修改 `when` 类似于 `when = when + period`，然后重新在 `P` 的四叉堆中找到位置。而如果 `period` 字段为0，则定时器执行完毕会被直接删除。

## 总结
因为存在跨 `P` 运行定时器的情况，因此定时器实现的复杂度稍微高了一点，整体设计通过四叉堆维护定时器，直接访问最早的定时器，获取最早定时器的触发时间点，并通过唤醒阻塞 `netpoll` 的线程来进行及时触发运行定时器。
