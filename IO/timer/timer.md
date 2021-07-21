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
	period int64
	f      func(interface{}, uintptr) // timer到期执行的函数
	arg    interface{} // f的参数
	seq    uintptr //
	nextwhen int64 // timer被修改后的时间
	status uint32 // timer的状态
}
```
### 2.1 addtimer
`addtimer` 是所有定时器创建的核心函数：
```
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
	doaddtimer(pp, t)
	unlock(&pp.timersLock)

	wakeNetPoller(when)
}
```
