### time

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-timer/#%E8%BF%90%E8%A1%8C%E8%AE%A1%E6%97%B6%E5%99%A8



http://xiaorui.cc/archives/6483

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
状态						解释
timerNoStatus			还没有设置状态
timerWaiting			等待触发
timerRunning			运行计时器函数
timerDeleted			被删除
timerRemoving			正在被删除
timerRemoved			已经被停止并从堆中删除
timerModifying			正在被修改
timerModifiedEarlier	被修改到了更早的时间
timerModifiedLater		被修改到了更晚的时间
timerMoving				已经被修改正在被移动

上述表格已经展示了不同状态的含义，但是我们还需要展示一些重要的信息，例如状态的存在时间、计时器是否在堆上等：

timerRunning、timerRemoving、timerModifying 和 timerMoving — 停留的时间都比较短；
timerWaiting、timerRunning、timerDeleted、timerRemoving、timerModifying、timerModifiedEarlier、timerModifiedLater 和 timerMoving — 计时器在处理器的堆上；
timerNoStatus 和 timerRemoved — 计时器不在堆上；
timerModifiedEarlier 和 timerModifiedLater — 计时器虽然在堆上，但是可能位于错误的位置上，需要重新排序；

// Ticket持有一个channel，定时往这个通道往送消息
type Ticker struct {
   C <-chan Time // The channel on which the ticks are delivered.
   r runtimeTimer
}

// 同runtime.timer同步
type runtimeTimer struct {
	//挂在的p
	pp puintptr
	//当前计时器被唤醒的时间
	when   int64
    //两次被唤醒的间隔
	period int64
    //每当计时器被唤醒时都会调用的函数
	f      func(interface{}, uintptr)
    //计时器被唤醒时调用 f 传入的参数
	arg    interface{}
	seq    uintptr
	//  计时器处于 timerModifiedXX 状态时，用于设置 when 字段；.
	nextwhen int64
	// 计时器的状态；
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
// addtimer增加一个timer到当前的p
func addtimer(t *timer) {
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
    //清理timer堆
	cleantimers(pp)
    //timer加入堆
	doaddtimer(pp, t)
	unlock(&pp.timersLock)
	//唤醒网络轮询器中休眠的线程
    当新添加的定时任务when小于netpoll等待的时间，那么wakeNetPoller会激活NetPoll的等待。激活的方法很简单，在findrunnable里的最后会使用超时阻塞的方法调用epollwait，这样既可监控了epfd红黑树上的fd，又可兼顾最近的定时任务的等待。
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
            // reset调整过的timer
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

func stopTimer(t *timer) bool {
	return deltimer(t)
}

// deltimer删除timer，也许它在其他的p上，因此我们不能实际上从堆上删除，我们只能标记它已经删除
// 它将会在适当的时间被它所在的P移除
// 返回计时器是否被删除
//runtime.deltimer 函数会标记需要删除的计时器，它会根据以下的规则处理计时器：
//timerWaiting -> timerModifying -> timerDeleted
//timerModifiedEarlier -> timerModifying -> timerDeleted
//timerModifiedLater -> timerModifying -> timerDeleted
//其他状态 -> 等待状态改变或者直接返回
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


// Reset stops a ticker and resets its period to the specified duration.
// The next tick will arrive after the new period elapses.
func (t *Ticker) Reset(d Duration) {
	if t.r.f == nil {
		panic("time: Reset called on uninitialized Ticker")
	}
	modTimer(&t.r, when(d), int64(d), t.r.f, t.r.arg, t.r.seq)
}

// modtimer modifies an existing timer.
// This is called by the netpoll code or time.Ticker.Reset.
// Reports whether the timer was modified before it was run.
//runtime.modtimer 会修改已经存在的计时器，它会根据以下的规则处理计时器：
//timerWaiting -> timerModifying -> timerModifiedXX
//timerModifiedXX -> timerModifying -> timerModifiedYY
//timerNoStatus -> timerModifying -> timerWaiting
//timerRemoved -> timerModifying -> timerWaiting
//timerDeleted -> timerModifying -> timerWaiting
//其他状态 -> 等待状态改变
func modtimer(t *timer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr) bool {
	if when < 0 {
		when = maxWhen
	}

	status := uint32(timerNoStatus)
	wasRemoved := false
	var pending bool
	var mp *m
loop:
	for {
		switch status = atomic.Load(&t.status); status {
		case timerWaiting, timerModifiedEarlier, timerModifiedLater:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()
			if atomic.Cas(&t.status, status, timerModifying) {
				pending = true // timer not yet run
				break loop
			}
			releasem(mp)
		case timerNoStatus, timerRemoved:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()

			// Timer was already run and t is no longer in a heap.
			// Act like addtimer.
			if atomic.Cas(&t.status, status, timerModifying) {
				wasRemoved = true
				pending = false // timer already run or stopped
				break loop
			}
			releasem(mp)
		case timerDeleted:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()
			if atomic.Cas(&t.status, status, timerModifying) {
				atomic.Xadd(&t.pp.ptr().deletedTimers, -1)
				pending = false // timer already stopped
				break loop
			}
			releasem(mp)
		case timerRunning, timerRemoving, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield()
		case timerModifying:
			// Multiple simultaneous calls to modtimer.
			// Wait for the other call to complete.
			osyield()
		default:
			badTimer()
		}
	}

	t.period = period
	t.f = f
	t.arg = arg
	t.seq = seq

	if wasRemoved {
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
		// The timer is in some other P's heap, so we can't change
		// the when field. If we did, the other P's heap would
		// be out of order. So we put the new when value in the
		// nextwhen field, and let the other P set the when field
		// when it is prepared to resort the heap.
		t.nextwhen = when

		newStatus := uint32(timerModifiedLater)
		if when < t.when {
			newStatus = timerModifiedEarlier
		}

		// Update the adjustTimers field.  Subtract one if we
		// are removing a timerModifiedEarlier, add one if we
		// are adding a timerModifiedEarlier.
		adjust := int32(0)
		if status == timerModifiedEarlier {
			adjust--
		}
		if newStatus == timerModifiedEarlier {
			adjust++
		}
		if adjust != 0 {
			atomic.Xadd(&t.pp.ptr().adjustTimers, adjust)
		}

		// Set the new status of the timer.
		if !atomic.Cas(&t.status, timerModifying, newStatus) {
			badTimer()
		}
		releasem(mp)

		// If the new status is earlier, wake up the poller.
		if newStatus == timerModifiedEarlier {
			wakeNetPoller(when)
		}
	}

	return pending
}
//runtime.adjusttimers 与 runtime.cleantimers 的作用相似，它们都会删除堆中的计时器并修改状态为 timerModifiedEarlier 和 timerModifiedLater 的计时器的时间，它们也会遵循相同的规则处理计时器状态：
//timerDeleted -> timerRemoving -> timerRemoved
//timerModifiedXX -> timerMoving -> timerWaiting
//与 runtime.cleantimers 不同的是，上述函数可能会遍历处理器堆中的全部计时器（包含退出条件），而不是只修改四叉堆顶部。
func adjusttimers(pp *p, now int64) {
	var moved []*timer
loop:
	for i := 0; i < len(pp.timers); i++ {
		t := pp.timers[i]
		switch s := atomic.Load(&t.status); s {
		case timerDeleted:
			// 删除堆中的计时器
		case timerModifiedEarlier, timerModifiedLater:
			// 修改计时器的时间
		case ...
		}
	}
	if len(moved) > 0 {
		addAdjustedTimers(pp, moved)
	}
}


// runtimer examines the first timer in timers. If it is ready based on now,
// it runs the timer and removes or updates it.
// Returns 0 if it ran a timer, -1 if there are no more timers, or the time
// when the first timer should run.
// The caller must have locked the timers for pp.
// If a timer is run, this will temporarily unlock the timers.
//go:systemstack


schedule findrunableg checktimer会调用
func runtimer(pp *p, now int64) int64 {
	for {
		t := pp.timers[0]
		if t.pp.ptr() != pp {
			throw("runtimer: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerWaiting:
			if t.when > now {
				// Not ready to run.
				return t.when
			}

			if !atomic.Cas(&t.status, s, timerRunning) {
				continue
			}
			// Note that runOneTimer may temporarily unlock
			// pp.timersLock.
			runOneTimer(pp, t, now)
			return 0

		case timerDeleted:
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			dodeltimer0(pp)
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				badTimer()
			}
			atomic.Xadd(&pp.deletedTimers, -1)
			if len(pp.timers) == 0 {
				return -1
			}

		case timerModifiedEarlier, timerModifiedLater:
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			t.when = t.nextwhen
			dodeltimer0(pp)
			doaddtimer(pp, t)
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}

		case timerModifying:
			// Wait for modification to complete.
			osyield()

		case timerNoStatus, timerRemoved:
			// Should not see a new or inactive timer on the heap.
			badTimer()
		case timerRunning, timerRemoving, timerMoving:
			// These should only be set when timers are locked,
			// and we didn't do it.
			badTimer()
		default:
			badTimer()
		}
	}
}


    

```





## 触发计时器 

- 调度器调度时会检查处理器中的计时器是否准备就绪；
- 系统监控会检查是否有未执行的到期计时器；

[`runtime.checkTimers`](https://draveness.me/golang/tree/runtime.checkTimers) 是调度器用来运行处理器中计时器的函数，它会在发生以下情况时被调用：

- 调度器调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 执行调度时；
- 调度器调用 [`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 获取可执行的 Goroutine 时；
- 调度器调用 [`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 从其他处理器窃取计时器时；



```go
// checkTimers runs any timers for the P that are ready.
// If now is not 0 it is the current time.
// It returns the current time or 0 if it is not known,
// and the time when the next timer should run or 0 if there is no next timer,
// and reports whether it ran any timers.
// If the time when the next timer should run is not 0,
// it is always larger than the returned time.
// We pass now in and out to avoid extra calls of nanotime.
//go:yeswritebarrierrec

如果处理器中不存在需要调整的计时器；
当没有需要执行的计时器时，直接返回；
当下一个计时器没有到期并且需要删除的计时器较少时都会直接返回；
如果处理器中存在需要调整的计时器，会调用 runtime.adjusttimers

调整了堆中的计时器之后，会通过 runtime.runtimer 依次查找堆中是否存在需要执行的计时器：

如果存在，直接运行计时器；
如果不存在，获取最新计时器的触发时间；

在 runtime.checkTimers 的最后，如果当前 Goroutine 的处理器和传入的处理器相同，并且处理器中删除的计时器是堆中计时器的 1/4 以上，就会调用 runtime.clearDeletedTimers 删除处理器全部被标记为 timerDeleted 的计时器，保证堆中靠后的计时器被删除。

runtime.clearDeletedTimers 能够避免堆中出现大量长时间运行的计时器，该函数和 runtime.moveTimers 也是唯二会遍历计时器堆的函数。
func checkTimers(pp *p, now int64) (rnow, pollUntil int64, ran bool) {
   // If there are no timers to adjust, and the first timer on
   // the heap is not yet ready to run, then there is nothing to do.
   if atomic.Load(&pp.adjustTimers) == 0 {
      next := int64(atomic.Load64(&pp.timer0When))
      if next == 0 {
         return now, 0, false
      }
      if now == 0 {
         now = nanotime()
      }
      if now < next {
         // Next timer is not ready to run.
         // But keep going if we would clear deleted timers.
         // This corresponds to the condition below where
         // we decide whether to call clearDeletedTimers.
         if pp != getg().m.p.ptr() || int(atomic.Load(&pp.deletedTimers)) <= int(atomic.Load(&pp.numTimers)/4) {
            return now, next, false
         }
      }
   }

   lock(&pp.timersLock)

   adjusttimers(pp)

   rnow = now
   if len(pp.timers) > 0 {
      if rnow == 0 {
         rnow = nanotime()
      }
      for len(pp.timers) > 0 {
         // Note that runtimer may temporarily unlock
         // pp.timersLock.
         if tw := runtimer(pp, rnow); tw != 0 {
            if tw > 0 {
               pollUntil = tw
            }
            break
         }
         ran = true
      }
   }

   // If this is the local P, and there are a lot of deleted timers,
   // clear them out. We only do this for the local P to reduce
   // lock contention on timersLock.
   if pp == getg().m.p.ptr() && int(atomic.Load(&pp.deletedTimers)) > len(pp.timers)/4 {
      clearDeletedTimers(pp)
   }

   unlock(&pp.timersLock)

   return rnow, pollUntil, ran
}


/ runOneTimer runs a single timer.
// The caller must have locked the timers for pp.
// This will temporarily unlock the timers while running the timer function.
//go:systemstack
    
根据计时器的 period 字段，上述函数会做出不同的处理：

如果 period 字段大于 0；
修改计时器下一次触发的时间并更新其在堆中的位置；
将计时器的状态更新至 timerWaiting；
调用 runtime.updateTimer0When 函数设置处理器的 timer0When 字段；
如果 period 字段小于或者等于 0；
调用 runtime.dodeltimer0 函数删除计时器；
将计时器的状态更新至 timerNoStatus；
更新计时器之后，上述函数会运行计时器中存储的函数并传入触发时间等参数。
func runOneTimer(pp *p, t *timer, now int64) {

	//定时器 回调方法
	f := t.f
	arg := t.arg
	seq := t.seq

	if t.period > 0 {
		// Leave in heap but adjust next time to fire.
		delta := t.when - now
		t.when += t.period * (1 + -delta/t.period)
        // 重排定时器
		siftdownTimer(pp.timers, 0)
		if !atomic.Cas(&t.status, timerRunning, timerWaiting) {
			badTimer()
		}
		updateTimer0When(pp)
	} else {
		// Remove from heap.
		dodeltimer0(pp)
		if !atomic.Cas(&t.status, timerRunning, timerNoStatus) {
			badTimer()
		}
	}
	unlock(&pp.timersLock)
	//调用方法
	f(arg, seq)
	lock(&pp.timersLock)
}




系统监控 #
系统监控函数 runtime.sysmon 也可能会触发函数的计时器，下面的代码片段中省略了大量与计时器无关的代码：
func sysmon() {
	...
	for {
		...
		now := nanotime()
		next, _ := timeSleepUntil()
		...
		lastpoll := int64(atomic.Load64(&sched.lastpoll))
		if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
			atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
			list := netpoll(0)
			if !list.empty() {
				incidlelocked(-1)
				injectglist(&list)
				incidlelocked(1)
			}
		}
		if next < now {
			startm(nil, false)
		}
		...
}
```