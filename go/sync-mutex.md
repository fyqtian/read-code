 sync.Mutex

https://mp.weixin.qq.com/s/7OZ8lmkiQU16884owQR-fw 信号量

https://mp.weixin.qq.com/s/IIrFNWMkY2sArWQ1R7i51g



- 正常模式（非公平模式）

阻塞等待的`goroutine`保存在`FIFO`的队列中，唤醒的`goroutine`不直接拥有锁，需要与新来的`goroutine`竞争获取锁。因为新来的`goroutine`很多已经占有了`CPU`，所以唤醒的`goroutine`在竞争中很容易输；但如果一个`goroutine`获取锁失败超过**`1ms`**,则会将`Mutex`切换为饥饿模式。



- 饥饿模式（公平模式）

这种模式下，直接将等待队列队头`goroutine`解锁`goroutine`；新来的`gorountine`也**不会尝试获得锁**，**而是直接插入到等待队列队尾。**

如果一个`goroutine`获得了锁，并且他在**等待队列队尾 或者 他等待小于`1ms`**，则会将`Mutex`的模式切换回正常模式。正常模式有更好的性能，新来的`goroutine`通过几次竞争可以直接获取到锁，尽管当前仍有等待的`goroutine`。而饥饿模式则是对正常模式的补充，防止等待队列中的`goroutine`永远没有机会获取锁。

![starv-mode](..\images\starv-mode.webp)

```go
//example
func main() {
   var m sync.Mutex
   var a int
   go func() {
      m.Lock()
      a++
      m.Unlock()
   }()
   m.Lock()
   a++
   fmt.Println(a)
   m.Unlock()
}
//output
1
```



```go
//Mutex是一个互斥锁
//Mutex空值时未锁
//Mutex在第一次使用后不能被拷贝
type Mutex struct {
   state int32
   sema  uint32
}

const (
	mutexLocked = 1 << iota // 互斥锁锁定
	mutexWoken  //从正常模式被从唤醒
	mutexStarving //饥饿模式 
	mutexWaiterShift = iota //等待锁得数量得goruntine
)

mutex.state & mutexLocked 加锁状态 1 表示已加锁 0 表示未加锁

mutex.state & mutexWoken  唤醒状态 1 表示已唤醒状态 0 表示未唤醒

mutex.state & mutexStarving  饥饿状态  1 表示饥饿状态 0表示正常状态

mutex.state >> mutexWaiterShift得到当前goroutine数目


正常模式下所有等待得goroutine按照顺序等待，唤醒得goruntine不会直接持有锁，而是和新请求得goroutine竞争
新请求锁得goroutine具有优势，它正在cpu上执行，刚唤醒得goroutine有很大概率失败，在这种情况下，这个被唤醒得goroutine会
加入道队列前面，如果一个等待得goroutine超过1ms没有获取锁它就会将锁转变为饥饿模式

饥饿模式下 锁得所有权将从unlock得goroutine直接交给等待队列得第一个，新来得goroutine不会尝试获取锁

如果一个等待得goroutine获取了锁并且满足以下条件它会将锁转为正常锁
1）队列中最后一个
2）等待时间小于1ms

自旋条件
mutex已经被locked了，处于正常模式下；
前 Goroutine 为了获取该锁进入自旋的次数小于四次；
当前机器CPU核数大于1；
当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空；

计算最新的new之后，CAS更新，如果更新成功且old状态是未被锁状态，并且锁不处于饥饿状态，就代表当前goroutine竞争成功并获取到了锁返回。(这也就是当前goroutine在正常模式下竞争时更容易获得锁的原因)

如果当前goroutine竞争失败，会调用 sync.runtime_SemacquireMutex 使用信号量保证资源不会被两个 Goroutine 获取。sync.runtime_SemacquireMutex 会在方法中不断调用尝试获取锁并休眠当前 Goroutine 等待信号量的释放，一旦当前 Goroutine 可以获取信号量，它就会立刻返回，sync.Mutex.Lock 方法的剩余代码也会继续执行。

(2) 饥饿模式
饥饿模式本身是为了一定程度保证公平性而设计的模式。所以饥饿模式不会有自旋的操作，新的 Goroutine 在该状态下不能获取锁、也不会进入自旋状态，它们只会在队列的末尾等待。

不管是正常模式还是饥饿模式，获取信号量，它就会从阻塞中立刻返回，并执行剩下代码：

在正常模式下，这段代码会设置唤醒和饥饿标记、重置迭代次数并重新执行获取锁的循环；
在饥饿模式下，当前 Goroutine 会获得互斥锁，如果等待队列中只存在当前 Goroutine，互斥锁还会从饥饿模式中退出；



进入饥饿模式 需要g被唤醒等待时间超过1毫秒，然后这期间正好有新的g抢占了锁，前一个g才会把锁标记为饥饿模式
unlock会把锁标记为唤醒
//Lock锁定m
//如果lock已经被使用，调用得goroutine阻塞直到mutex可用
func (m *Mutex) Lock() {
	// 原子操作 如果m.state是0写入1 成功返回
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	m.lockSlow()
}

32                                               3             2             1             0 
 |                                               |             |             |             | 
 |                                               |             |             |             | 
 v-----------------------------------------------v-------------v-------------v-------------+ 
 |                                               |             |             |             v 
 |                 waitersCount                  |mutexStarving| mutexWoken  | mutexLocked | 
 |                                               |             |             |             | 
 +-----------------------------------------------+-------------+-------------+-------------+     

func (m *Mutex) lockSlow() {
	var waitStartTime int64 //当前等待时间
	starving := false		//饥饿模式
	awoke := false			//唤醒状态
	iter := 0 				//自选次数
	old := m.state			//当前状态
	for {
        //饥饿模式不做自旋，当前锁得持有者直接唤醒第一个等待得
        //尝试自旋
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            //设置awoke 标记状态唤醒中
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
        //state可能得状态
        //锁未释放 正常状态
        //锁未释放 饥饿状态
        //锁释放 正常状态
        //锁释放 饥饿状态
        //当前awoke可能是true 可能是false(其他gorutine设置了awoken状态)
        
		new := old
		//如果未处在饥饿状态 标记上锁
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
        //如果旧状态未标记上锁和饥饿 标记等待goroutine+1
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
        
        //如果当前goroutine为饥饿模式并且已经上锁 锁标记为饥饿
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
            //如果当前为goroutine为唤醒 取消唤醒标记
            //因为当前goroutine要么获得锁要么进入休眠
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
        //设置锁新状态，
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
            //如果旧锁未锁定并且未饥饿
            //获取锁得所有权返回
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// 计算等待时间
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
            // 未获取到锁sleep阻塞
            // 如果是新加入得goroutine queueLifo==false加入等待队列微博
            // 如果是唤醒过得goroutine queueLifo==true加入等待队列头部
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            //sleep后唤醒
            //已经饥饿或者 等待时间大于1毫秒
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			//当前状态
            old = m.state
            //如果当前锁处在饥饿状态
            //那么锁应该未锁定 锁直接交给当前goroutine
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
                //状态不一致
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
                
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
                //锁获取时间小于1秒或者在是队列最后一个
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					//等待数量减1
                    delta -= mutexStarving
				}
    
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
            //状态变更过 从头再来
			old = m.state
		}
	}


}

// Active spinning for sync.Mutex.
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
	// sync.Mutex is cooperative, so we are conservative with spinning.
	// Spin only few times and only if running on a multicore machine and
	// GOMAXPROCS>1 and there is at least one other running P and local runq is empty.
	// As opposed to runtime mutex we don't do passive spinning here,
	// because there can be work on global runq or on other Ps.
	if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}

//go:linkname sync_runtime_doSpin sync.runtime_doSpin
//go:nosplit
func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}

//go:linkname sync_runtime_SemacquireMutex sync.runtime_SemacquireMutex
func sync_runtime_SemacquireMutex(addr *uint32, lifo bool, skipframes int) {
	semacquire1(addr, lifo, semaBlockProfile|semaMutexProfile, skipframes)
}


func semacquire1(addr *uint32, lifo bool, profile semaProfileFlags, skipframes int) {
	gp := getg()
	if gp != gp.m.curg {
		throw("semacquire not on the G stack")
	}

	// Easy case.
	if cansemacquire(addr) {
		return
	}

	// Harder case:
	//	increment waiter count
	//	try cansemacquire one more time, return if succeeded
	//	enqueue itself as a waiter
	//	sleep
	//	(waiter descriptor is dequeued by signaler)
	s := acquireSudog()
	root := semroot(addr)
	t0 := int64(0)
	s.releasetime = 0
	s.acquiretime = 0
	s.ticket = 0
	if profile&semaBlockProfile != 0 && blockprofilerate > 0 {
		t0 = cputicks()
		s.releasetime = -1
	}
	if profile&semaMutexProfile != 0 && mutexprofilerate > 0 {
		if t0 == 0 {
			t0 = cputicks()
		}
		s.acquiretime = t0
	}
	for {
		lockWithRank(&root.lock, lockRankRoot)
		// Add ourselves to nwait to disable "easy case" in semrelease.
        递增nwait进而避免semrelease中的快速路径
		atomic.Xadd(&root.nwait, 1)
		// Check cansemacquire to avoid missed wakeup.
		if cansemacquire(addr) {
			atomic.Xadd(&root.nwait, -1)
			unlock(&root.lock)
			break
		}
		// Any semrelease after the cansemacquire knows we're waiting
		// (we set nwait above), so go to sleep.
		root.queue(addr, s, lifo)
		goparkunlock(&root.lock, waitReasonSemacquire, traceEvGoBlockSync, 4+skipframes)
		if s.ticket != 0 || cansemacquire(addr) {
			break
		}
	}
	if s.releasetime > 0 {
		blockevent(s.releasetime-t0, 3+skipframes)
	}
	releaseSudog(s)
}


unlock完毕mutex.state!=0 则存在以下可能
直接将锁交给等待队列的第一个goroutine
当前存在等待goroutine 然后唤醒它 但不是第一个goroutine
当前存在自旋等待的goroutine 则不唤醒其他等待gorotune
正常模式下
饥饿模式下

func (m *Mutex) Unlock() {
	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
    //如果等于0解锁成功 说明没有其他人等待锁
	if new != 0 {
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
    //unlock未上锁得锁
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
    //非饥饿状态
	if new&mutexStarving == 0 {
		old := new
		for {
            //如果互斥锁不存在等待者或者互斥锁的
            //或者已经有gorutine唤醒或者已经有goroutine获取锁或者进入饥饿模式
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
            //如果互斥锁存在等待者，会通过 sync.runtime_Semrelease 唤醒等待者并移交锁的所有权；
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
        //在饥饿模式下，调用runtime_Semrelease 方法将当前锁交给下一个正在尝试获取锁的等待者，等待者被唤醒后会得到锁，在这时互斥锁还不会退出饥饿状态
		runtime_Semrelease(&m.sema, true, 1)
	}
}

```







### 信号量

`Mutex`对`goroutine`的阻塞和唤醒操作是利用`semaphore`来实现的，大致的思路是：Go runtime维护了一个全局的变量`semtable`,它保持了所有的信号量

```go
const semTabSize = 251

var semtable [semTabSize]struct {
 root semaRoot
 pad  [cpu.CacheLinePadSize - unsafe.Sizeof(semaRoot{})]byte
}
```

每个信号量都由一个变量地址指定，Mutex的栗子里就是`mutex.sema`的地址

```go
type semaRoot struct {
 lock  mutex
 treap *sudog // root of balanced tree of unique waiters.
 nwait uint32 // Number of waiters. Read w/o the lock.
}
```

<img src="D:\gocode\read-code\images\sema-table.webp" alt="sema-table" style="zoom:67%;" />



```go

func semroot(addr *uint32) *semaRoot {
    // 如果定位到同一个key？
	return &semtable[(uintptr(unsafe.Pointer(addr))>>3)%semTabSize].root
}

func semacquire1(addr *uint32, lifo bool, profile semaProfileFlags, skipframes int) {
   gp := getg()
   if gp != gp.m.curg {
      throw("semacquire not on the G stack")
   }

 // 低成本case
  // 若addr大于1 并通过CAS -1 成功，则获取信号量成功 不需要阻塞
   if cansemacquire(addr) {
      return
   }
 // 复杂 case:
 // 增加等待goroutine数量
 // 再次尝试cansemacquire 成功则返回
 // 失败则将自己作为一个waiter入队
 // sleep
 // (waiter descriptor is dequeued by signaler)
    从当前p.sudogcache拿 当前p没有从全局拿 没有就新建
   s := acquireSudog()
   root := semroot(addr)
   t0 := int64(0)
   s.releasetime = 0
   s.acquiretime = 0
   s.ticket = 0
   if profile&semaBlockProfile != 0 && blockprofilerate > 0 {
      t0 = cputicks()
      s.releasetime = -1
   }
   if profile&semaMutexProfile != 0 && mutexprofilerate > 0 {
      if t0 == 0 {
         t0 = cputicks()
      }
      s.acquiretime = t0
   }
   for {
      lockWithRank(&root.lock, lockRankRoot)
        // 给nwait+1 这样semrelease中不会进低成本路径了
      atomic.Xadd(&root.nwait, 1)
      // 检查 cansemacquire 避免错过唤醒
      if cansemacquire(addr) {
         atomic.Xadd(&root.nwait, -1)
         unlock(&root.lock)
         break
      }
         //cansemacquire之后的semrelease都可以知道我们正在等待
    //上面设置了nwait，所以会直接进入sleep 即goparkunlock
      root.queue(addr, s, lifo)
      goparkunlock(&root.lock, waitReasonSemacquire, traceEvGoBlockSync, 4+skipframes)
      if s.ticket != 0 || cansemacquire(addr) {
         break
      }
   }
   if s.releasetime > 0 {
      blockevent(s.releasetime-t0, 3+skipframes)
   }
   releaseSudog(s)
}




func semrelease1(addr *uint32, handoff bool, skipframes int) {
	root := semroot(addr)
	atomic.Xadd(addr, 1)

	 // Easy case: no waiters?
 // 这个检查必须发生在xadd之后 避免错过唤醒
 // (see loop in semacquire).
	if atomic.Load(&root.nwait) == 0 {
		return
	}

	 // Harder case: 搜索一个waiter 并唤醒它
	lockWithRank(&root.lock, lockRankRoot)
	if atomic.Load(&root.nwait) == 0 {
		  // count值已经被另一个goroutine消费了   （这不重复解锁吗）
 		 // 所以不需要唤醒其他goroutine      
		unlock(&root.lock)
		return
	}
	s, t0 := root.dequeue(addr)
	if s != nil {
		atomic.Xadd(&root.nwait, -1)
	}
	unlock(&root.lock)
	if s != nil { // May be slow or even yield, so unlock first
		acquiretime := s.acquiretime
		if acquiretime != 0 {
			mutexevent(t0-acquiretime, 3+skipframes)
		}
		if s.ticket != 0 {
			throw("corrupted semaphore ticket")
		}
		if handoff && cansemacquire(addr) {
			s.ticket = 1
		}
		readyWithTime(s, 5+skipframes)
		if s.ticket == 1 && getg().m.locks == 0 {
			// Direct G handoff
			// readyWithTime has added the waiter G as runnext in the
			// current P; we now call the scheduler so that we start running
			// the waiter G immediately.
			// Note that waiter inherits our time slice: this is desirable
			// to avoid having a highly contended semaphore hog the P
			// indefinitely. goyield is like Gosched, but it emits a
			// "preempted" trace event instead and, more importantly, puts
			// the current G on the local runq instead of the global one.
			// We only do this in the starving regime (handoff=true), as in
			// the non-starving case it is possible for a different waiter
			// to acquire the semaphore while we are yielding/scheduling,
			// and this would be wasteful. We wait instead to enter starving
			// regime, and then we start to do direct handoffs of ticket and
			// P.
			// See issue 33747 for discussion.
			goyield()
		}
	}
}

```