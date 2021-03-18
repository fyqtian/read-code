### RWMutex



```go
// There is a modified copy of this file in runtime/rwmutex.go.
// If you make any changes here, see if you should make them there.

// A RWMutex is a reader/writer mutual exclusion lock.
// The lock can be held by an arbitrary number of readers or a single writer.
// The zero value for a RWMutex is an unlocked mutex.
//
// A RWMutex must not be copied after first use.
//
// If a goroutine holds a RWMutex for reading and another goroutine might
// call Lock, no goroutine should expect to be able to acquire a read lock
// until the initial read lock is released. In particular, this prohibits
// recursive read locking. This is to ensure that the lock eventually becomes
// available; a blocked Lock call excludes new readers from acquiring the
// lock.
// RWMutex是读和写的互斥锁
// 锁能被任意数量的读者或者一个写者持有
// 对于空值的RWMutex是一个未上锁的锁
// 如果一个goroutine持有RWMutex准备读，其他的goroutine也许会调用Lock，没有goroutine应该期盼能够获取一个读锁
// 直到初始的读锁被释放，特别是，这禁止递归读取锁定。这是为了确保锁最终变成可用；阻止的锁调用将新的读卡器排除在获取锁

type RWMutex struct {
   w           Mutex  // held if there are pending writers
   writerSem   uint32 // semaphore for writers to wait for completing readers
   readerSem   uint32 // semaphore for readers to wait for completing writers
   readerCount int32  // number of pending readers
   readerWait  int32  // number of departing readers
}

// RLock locks rw for reading.
//
// It should not be used for recursive read locking; a blocked Lock
// call excludes new readers from acquiring the lock. See the
// documentation on the RWMutex type.
func (rw *RWMutex) RLock() {
    //readerCount <0 说明有写锁 挂起等待
 	// 写锁的时候 会把这个值 减去 1<<30
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		// A writer is pending, wait for it.
        // 挂起等待信号量 唤醒
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}

// RUnlock undoes a single RLock call;
// it does not affect other simultaneous readers.
// It is a run-time error if rw is not locked for reading
// on entry to RUnlock.
func (rw *RWMutex) RUnlock() {
    // r < 0 有写锁在等待
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		// Outlined slow-path to allow the fast-path to be inlined
		rw.rUnlockSlow(r)
	}
}


func (rw *RWMutex) rUnlockSlow(r int32) {
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		race.Enable()
		throw("sync: RUnlock of unlocked RWMutex")
	}
	// A writer is pending.
    // 读锁全部释放了 唤醒写锁
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		// The last reader unblocks the writer.
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}

// Lock locks rw for writing.
// If the lock is already locked for reading or writing,
// Lock blocks until the lock is available.
func (rw *RWMutex) Lock() {
    //上互斥锁
	rw.w.Lock()
	// Announce to readers there is a pending writer.
   	
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.
    // 如果 r != 0 说明有读者占用读锁 等待读锁释放
    // rw.readerWait 等待几个读锁释放
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
}


// Unlock unlocks rw for writing. It is a run-time error if rw is
// not locked for writing on entry to Unlock.
//
// As with Mutexes, a locked RWMutex is not associated with a particular
// goroutine. One goroutine may RLock (Lock) a RWMutex and then
// arrange for another goroutine to RUnlock (Unlock) it.
func (rw *RWMutex) Unlock() {
	// Announce to readers there is no active writer.
    // r = 等待的读锁
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}
	// 唤醒所有的读者
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	// Allow other writers to proceed.
	rw.w.Unlock()
}

```