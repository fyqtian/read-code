### sync.Cond

```go
// example
func main() {
   var m sync.Mutex
   cond := sync.NewCond(&m)
   go func() {
      for {
         <-time.Tick(1e9)
         cond.Signal()
      }

   }()
   for {
      cond.L.Lock()
      cond.Wait()
      log.Println("receive")
      cond.L.Unlock()
   }
}
//output
2021/01/26 15:59:20 receive
2021/01/26 15:59:21 receive
2021/01/26 15:59:22 receive
2021/01/26 15:59:23 receive
```



```go
// Cond实现了一个条件变量，一个等待或宣布事件发生的goroutines的集合点。
// 每个Cond都有一个相关的Locker L（通常是* Mutex或* RWMutex）
type Cond struct {
	noCopy noCopy
	L Locker
    // 通知列表,调用Wait()方法的goroutine会被放入list中,每次唤醒,从这里取出
    // notifyList对象，维护等待唤醒的goroutine队列,使用链表实现
	notify  notifyList
    // 复制检查,检查cond实例是否被复制
    // copyChecker对象，实际上是uintptr对象，保存自身对象地址
	checker copyChecker
}

// NewCond returns a new Cond with Locker l.
func NewCond(l Locker) *Cond {
   return &Cond{L: l}
}

// 唤醒一个等待的c上的goroutine
// 允许但需要调用者持有锁
func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}


// 等待原子解锁c.L并暂停执行调用goroutine。
// 稍后恢复执行后，Wait会在返回之前锁定c.L.
// 与其他系统不同，除非被广播或信号唤醒，否则等待无法返回。
// 因为等待第一次恢复时c.L没有被锁定，
// 所以当Wait返回时，调用者通常不能认为条件为真。
// 相反，调用者应该循环等待：
//    c.L.Lock()
//    for !condition() {
//        c.Wait()
//    }
//    ... make use of condition ...
//    c.L.Unlock()
//
func (c *Cond) Wait() {
	c.checker.check()
     // 将当前goroutine加入等待队列, 该方法在 runtime 包的 notifyListAdd 函数中实现 
	t := runtime_notifyListAdd(&c.notify)
    // 释放锁, 因此在调用Wait方法前，必须保证获取到了cond的锁，否则会报错
	c.L.Unlock()
    // 等待队列中的所有的goroutine执行等待唤醒操作
    // 将当前goroutine挂起，等待唤醒信号
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}


// notifyListAdd将调用者加入notify list，以便他可以接收到通知，调用者最终必须调用notifyListWait等待通知
// 通过传递返回的ticketnumber
func notifyListAdd(l *notifyList) uint32 {
	return atomic.Xadd(&l.wait, 1) - 1
}

// notifyListWait等待通知，如果有调用了notifyListAdd立刻返回否则阻塞
func notifyListWait(l *notifyList, t uint32) {
	lockWithRank(&l.lock, lockRankNotifyList)

	// 如果已经有notify
	if less(t, l.notify) {
		unlock(&l.lock)
		return
	}

	// 当前G加入队列
	s := acquireSudog()
	s.g = getg()
	s.ticket = t
	s.releasetime = 0
	t0 := int64(0)
	。。。
    //更新列表
	if l.tail == nil {
		l.head = s
	} else {
		l.tail.next = s
	}
	l.tail = s
    //挂起阻塞
	goparkunlock(&l.lock, waitReasonSyncCondWait, traceEvGoBlockCond, 3)
	。。。
	releaseSudog(s)
}


// notifyListNotifyOne notifies one entry in the list.
func notifyListNotifyOne(l *notifyList) {
	//如果当前没有waiter
	if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
		return
	}
	lockWithRank(&l.lock, lockRankNotifyList)
    // re-check
	t := l.notify
	if t == atomic.Load(&l.wait) {
		unlock(&l.lock)
		return
	}
    // 更新下个通知的ticket number
	atomic.Store(&l.notify, t+1)

	// This scan looks linear but essentially always stops quickly.
	// Because g's queue separately from taking numbers,
	// there may be minor reorderings in the list, but we
	// expect the g we're looking for to be near the front.
	// The g has others in front of it on the list only to the
	// extent that it lost the race, so the iteration will not
	// be too long. This applies even when the g is missing:
	// it hasn't yet gotten to sleep and has lost the race to
	// the (few) other g's that we find on the list.
    // 尝试找到需要通知的g，如果还没在list无法找到。
    // 一旦看到新的通知number 将不会park
    // 这个扫描看上去是线性的，但是基本上停止很快
    // 因为g队列通过taking number分开
    // 列表中可能有一个最小记录的排序， 但是我们期盼
	for p, s := (*sudog)(nil), l.head; s != nil; p, s = s, s.next {
		if s.ticket == t {
			n := s.next
			if p != nil {
				p.next = n
			} else {
				l.head = n
			}
			if n == nil {
				l.tail = p
			}
			unlock(&l.lock)
			s.next = nil
			readyWithTime(s, 4)
			return
		}
	}
	unlock(&l.lock)
}
```