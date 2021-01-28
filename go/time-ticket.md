### time.Ticket

```go
func main() {
	t := time.NewTicker(1e9)
	i := 0
	for v := range t.C {
		if i == 3 {
			t.Stop()
			return
		}
		time.Sleep(3e9)
		log.Println(i, v)
		i++
	}
}

//output
2021/01/27 16:01:25 0 2021-01-27 16:01:22.0422331 +0800 CST m=+1.003463101
2021/01/27 16:01:28 1 2021-01-27 16:01:23.0435087 +0800 CST m=+2.004738701
2021/01/27 16:01:31 2 2021-01-27 16:01:26.0423314 +0800 CST m=+5.003561401
```



```go
// Ticket持有一个channel，定时往这个通道往送消息
type Ticker struct {
   C <-chan Time // The channel on which the ticks are delivered.
   r runtimeTimer
}

// 同runtime.timer同步
type runtimeTimer struct {
	// If this timer is on a heap, which P's heap it is on.
	// puintptr rather than *p to match uintptr in the versions
	// of this struct defined in other packages.
    // 如果timer在在堆上，PP指向p？
	pp puintptr
	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(arg, now) in the timer goroutine, so f must be
	// a well-behaved function and not block.
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr
	// What to set the when field to in timerModifiedXX status.
	nextwhen int64
	// The status field holds one of the values below.
	status uint32
}

// NewTicker返回一个新的Ticker，包含一个channel定时回发送时间
// 调整间隔或丢弃ticks对于慢接收者
// d必须大于0，否则panic
// stop ticker释放相关资源
func NewTicker(d Duration) *Ticker {
	if d <= 0 {
		panic(errors.New("non-positive interval for NewTicker"))
	}
    // 创建1个缓冲区的channel
    // 如果调用方消费落后，丢弃tick，直到调用方追上
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

// startTimer adds t to the timer heap.
//go:linkname startTimer time.startTimer
func startTimer(t *timer) {
	addtimer(t)
}

// addtimer adds a timer to the current P.
// This should only be called with a newly created timer.
// That avoids the risk of changing the when field of a timer in some P's heap,
// which could cause the heap to become unsorted.
// addtimer增加一个timer到当前的p
// 
func addtimer(t *timer) {
	// when must never be negative; otherwise runtimer will overflow
	// during its delta calculation and never expire other runtime timers.
	if t.when < 0 {
		t.when = maxWhen
	}
    //状态必须为0
	if t.status != timerNoStatus {
		throw("addtimer called with initialized timer")
	}
	t.status = timerWaiting
	when := t.when
    //当前P
	pp := getg().m.p.ptr()
	lock(&pp.timersLock)
    //清晰timer堆
	cleantimers(pp)
    //timer加入堆
	doaddtimer(pp, t)
	unlock(&pp.timersLock)

	wakeNetPoller(when)
}

// cleantimers清理堆上的timer,这回加速创建和删除timer,它们留再堆中回减慢addtimer,
// 报告是否发现timer问题被发现，调用者必须锁timer
func cleantimers(pp *p) {
	gp := getg()
	for {
		if len(pp.timers) == 0 {
			return
		}
        // 这个循环理论上可以运行一段时间，因为它持有了timerlock，它不能被抢占
        // 如果有人尝试抢占，返回，可以后期再清理
		if gp.preemptStop {
			return
		}
		
		t := pp.timers[0]
		if t.pp.ptr() != pp {
			throw("cleantimers: bad p")
		}
        //拿到状态
		switch s := atomic.Load(&t.status); s {
            //如果被删除 状态修改为移除中
		case timerDeleted:
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
            //从堆上移走 调整堆
			dodeltimer0(pp)
            //状态变更为已经移除
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
                // 状态变更失败 throw
				badTimer()
			}
			atomic.Xadd(&pp.deletedTimers, -1)
		case timerModifiedEarlier, timerModifiedLater:
            // timer修改了时间（？）
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			// 更新when
			t.when = t.nextwhen
            //堆上删除 重新插入
			dodeltimer0(pp)
			doaddtimer(pp, t)
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}
		default:
			// Head of timers does not need adjustment.
			return
		}
	}
}

// doaddtimer添加timer到当前的p上的堆
// 调用者必须持有p的锁
func doaddtimer(pp *p, t *timer) {
    // timer以来network poller,因此我们必须确保netpoller已经启动
	if netpollInited == 0 {
        //windows创建iocp linux创建Epoll
		netpollGenericInit()
	}

	if t.pp != 0 {
		throw("doaddtimer: P already set in timer")
	}
	t.pp.set(pp)
	i := len(pp.timers)
	pp.timers = append(pp.timers, t)
    //调整堆
	siftupTimer(pp.timers, i)
 	// 如果当前timer是最早唤醒timer
	if t == pp.timers[0] {
		atomic.Store64(&pp.timer0When, uint64(t.when))
	}
	atomic.Xadd(&pp.numTimers, 1)
}
// 唤醒正在netpoll休眠的线程，前提是when的值小于pollUntil时间。
func wakeNetPoller(when int64) {
	if atomic.Load64(&sched.lastpoll) == 0 {
		// In findrunnable we ensure that when polling the pollUntil
		// field is either zero or the time to which the current
		// poll is expected to run. This can have a spurious wakeup
		// but should never miss a wakeup.
		pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
		if pollerPollUntil == 0 || pollerPollUntil > when {
			netpollBreak()
		}
	}
}


// stopTimer stops a timer.
// It reports whether t was stopped before being run.
//go:linkname stopTimer time.stopTimer
func stopTimer(t *timer) bool {
	return deltimer(t)
}

// deltimer删除timer，也许它在其他的p上，因此我们不能实际上从堆上删除，我们只能标记它已经删除
// 它将会在适当的时间被它所在的P移除
// 返回计时器是否被删除
func deltimer(t *timer) bool {
	for {
        //加载状态
		switch s := atomic.Load(&t.status); s {
		case timerWaiting, timerModifiedLater:
            // 计时器处在timerModifying禁止抢占，可能导致死锁 See #38070.
			// m上锁
            mp := acquirem()
            // 更新状态
			if atomic.Cas(&t.status, s, timerModifying) {
                // 必须在变更状态前获取t.pp，作为另一个goroutine的timer清理工
                // 可以清理删除的timer？
				tpp := t.pp.ptr()
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
		case timerModifiedEarlier:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp := acquirem()
			if atomic.Cas(&t.status, s, timerModifying) {
				// Must fetch t.pp before setting status
				// to timerDeleted.
				tpp := t.pp.ptr()
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
			// Timer was already run.
			return false
		case timerRunning, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield()
		case timerNoStatus:
			// Removing timer that was never added or
			// has already been run. Also see issue 21874.
			return false
		case timerModifying:
			// Simultaneous calls to deltimer and modtimer.
			// Wait for the other call to complete.
			osyield()
		default:
			badTimer()
		}
	}
}

// NewTimer creates a new Timer that will send
// the current time on its channel after at least duration d.
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

func sendTime(c interface{}, seq uintptr) {
	// Non-blocking send of time on c.
	// Used in NewTimer, it cannot block anyway (buffer).
	// Used in NewTicker, dropping sends on the floor is
	// the desired behavior when the reader gets behind,
	// because the sends are periodic.
	select {
	case c.(chan Time) <- Now():
	default:
	}
}


func AfterFunc(d Duration, f func()) *Timer {
    t := &Timer{
        r: runtimeTimer{
            when: when(d),
            f:    goFunc,
            arg:  f,
        },
    }
    startTimer(&t.r)
    return t
}

func goFunc(arg interface{}, seq uintptr) {
	go arg.(func())()
}

```