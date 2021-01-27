### sync.WaitGroup



```go
func main() {
   wg := sync.WaitGroup{}
   wg.Add(100)
   for i := 0; i < 100; i++ {
      go func(i int) {
         fmt.Println(i)
         wg.Done()
      }(i)
   }
   wg.Wait()
}
```



```go
//Wait等待一批goroutine完成
//主线程调用Add设置goroutine的数量
//每个执行任务的线程运行并在任务结束调用Done
//Wait可以用来阻塞主线程并等所有任务完成
type WaitGroup struct {
   noCopy noCopy

   //用于存储计数器(counter)和waiter的值
  // 只需要64位,即8个字节,其中高32位是counter值,低32位值是waiter值
	// 不直接使用uint64,是因为uint64的原子操作需要64位系统,而32位系统下,可能会出现崩溃
	// 所以这里用byte数组来实现,32位系统下4字节对齐,64位系统下8字节对齐,所以申请12个字节,其中必定有8个字节是符合8字节对齐的,下面的state()函数中有进行判断
   state1 [3]uint32
}

// state returns pointers to the state and sema fields stored within wg.state1.
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    //如果是64位
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}

// Add adds delta, 可能是负数
// 如果counter是0，所有的goroutine阻塞等待wait释放
// 如果counter变成负数触发panic
// 注意调用者使用负数

func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	
    state := atomic.AddUint64(statep, uint64(delta)<<32)
    //高32位count
	v := int32(state >> 32)
    //低32位wait数量
	w := uint32(state)
	//counter小于0 非法
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
    // wait不等于0说明已经执行了Wait，此时不容许先wait再add
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
   // 正常情况，Add会让v增加，Done会让v减少，如果没有全部Done掉，此处v总是会大于0的，直到v为0才往下走
    // 而w代表是有多少个goruntine在等待done的信号，wait中通过compareAndSwap对这个w进行加1
    //count>0说明不需要释放信号量
    //w=0说明没有waiter
	if v > 0 || w == 0 {
		return
	}
    //下面是 counter == 0 并且 waiter > 0的情况
	//现在若原state和新的state不等，则有以下两种可能
	//1. Add 和 Wait方法同时调用
	//2. counter已经为0，但waiter值有增加，这种情况永远不会触发信号量了
	// 以上两种情况都是错误的，所以触发异常
	//注：state := atomic.AddUint64(statep, uint64(delta)<<32)  这一步调用之后，state和*statep的值应该是相等的，除非有以上两种情况发生
	 // 当v为0(Done掉了所有)或者w不为0(已经开始等待)才会到这里，但是在这个过程中又有一次Add，导致statep变化，panic
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	//将waiter 和 counter都置为0
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}


// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		w := uint32(state)
        //当前没counter
		if v == 0 {
			// Counter is 0, no need to wait.
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
		//waiter数量+1
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
            //等待信号量
			runtime_Semacquire(semap)
            //如果counter不为0
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			return
		}
	}
}

// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}

```