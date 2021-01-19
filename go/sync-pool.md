### sync.Pool



```go
https://zhuanlan.zhihu.com/p/133638023
https://louyuting.blog.csdn.net/article/details/90647884
https://www.cnblogs.com/luozhiyun/p/14194872.html

//example
package main

import (
   "fmt"
   "sync"
)

type object struct {
	name string
	buf  []byte
}

func main() {
	p := sync.Pool{New: func() interface{} {
		return &object{}
	}}

	o1 := p.Get().(*object)
	o1.name = "object 1"
	o2 := p.Get().(*object)
	fmt.Printf("%p------%p-----o1.name=%s\n", o1, o2, o1.name)

	o3 := p.Get().(*object)
	o3.name = "object 3"
	p.Put(o3)
	o4 := p.Get().(*object)
	fmt.Printf("%p------%p------o4.name=%s\n", o3, o4, o4.name)

	o5 := p.Get().(*object)
	o5.name = "object 5"
	p.Put(o5)
	runtime.GC()
	o6 := p.Get().(*object)
	fmt.Printf("%p------%p------o6.name=%s\n", o5, o6, o6.name)
}

//output
0xc00006e360------0xc00006e390-----o1.name=object 1
0xc00006e3c0------0xc00006e3c0------o4.name=object 3
0xc00006e3f0------0xc00006e360------o6.name= object 5(不确定 可能打印 可能不打印看是否会2次GC)
```

```go

//一个pool用来存放临时对象得集合，分别保存和取回
// 任何项目存储在pool在被移除得时候不会有任何通知，如果pool持有这个引用，对象将会被释放

// pool是并发安全
//pool的目的是用来缓存分配但未使用的对象将来复用
//减轻垃圾回收的压力，这样可以轻松构建有效的线程安全的列表，但是这不适合所有的(免费列表)

// An appropriate use of a Pool is to manage a group of temporary items
// silently shared among and potentially reused by concurrent independent
// clients of a package. Pool provides a way to amortize allocation overhead
// across many clients.
// 
// An example of good use of a Pool is in the fmt package, which maintains a
// dynamically-sized store of temporary output buffers. The store scales under
// load (when many goroutines are actively printing) and shrinks when
// quiescent.

// On the other hand, a free list maintained as part of a short-lived object is
// not a suitable use for a Pool, since the overhead does not amortize well in
// that scenario. It is more efficient to have such objects implement their own
// free list.
//
// A Pool must not be copied after first use.
type Pool struct {
   noCopy noCopy

   local     unsafe.Pointer // local,固定大小per-P池, 实际类型为 [P]poolLocal
   localSize uintptr        // local array 的大小

   victim     unsafe.Pointer // 上一轮gc存活
   victimSize uintptr        // 生存者大小

   New func() interface{}
}

// Local per-P Pool appendix.
type poolLocalInternal struct {
	private interface{} // P的私有缓存区
	shared  poolChain   //本地的P可以pushHead/popHead;其他P只能popTail.
}

type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

// Get从pool选择任意一个项目，从pool中移除返回给调用者
// Get也许会选择忽略pool
// 调用者不能假设和Put放入的value有任何关联
// 如果Get返回Nil，而p.New不为nil，那么返回p.New
func (p *Pool) Get() interface{} {
	l, pid := p.pin()
    //拿到私有对象
	x := l.private
	l.private = nil
    //如果私有对象不存在
	if x == nil {
        //从shard头部获取
		x, _ = l.shared.popHead()
		if x == nil {
            //未找到从其他P那边找
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
	//如果还未找到新建
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}


// pin绑定当前g和p，禁止抢占，从poolLocal池中返回对应的poolLocal
// Caller must call runtime_procUnpin() when done with the pool.
func (p *Pool) pin() (*poolLocal, int) {
    //当前G找到M M找到P
	pid := runtime_procPin()
	// In pinSlow we store to local and then to localSize, here we load in opposite order.
	// Since we've disabled preemption, GC cannot happen in between.
	// Thus here we must observe local at least as large localSize.
	// We can observe a newer/larger local, it is fine (we must observe its zero-initialized-ness).
	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	l := p.local                          // load-consume
    //如果当前的p小于size说明当前的p没有poolLocal
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	return p.pinSlow()
}

func indexLocal(l unsafe.Pointer, i int) *poolLocal {
	lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
	return (*poolLocal)(lp)
}

func (p *Pool) pinSlow() (*poolLocal, int) {
	// Retry under the mutex.
	// Can not lock the mutex while pinned.
	//解除pin
    runtime_procUnpin()
    //上全局锁
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	//再次pin住
    //这个期间可能会被调度
    pid := runtime_procPin()
	// poolCleanup won't be called while we are pinned.
	s := p.localSize
	l := p.local
    //再查找一次
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
    //把p存入全局
  	//gc的时候遍历清扫
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
    //当前P的数量
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	//没理解 每个p的local都不同怎么去取别人的poolLocal
    atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
    return &local[pid], pid
}

// Put adds x to the pool.
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	//获取localpool
	l, _ := p.pin()
    //优先放入private
	if l.private == nil {
		l.private = x
		x = nil
	}
    //放入sharded
	if x != nil {
		l.shared.pushHead(x)
	}
	runtime_procUnpin()
}


func (p *Pool) getSlow(pid int) interface{} {
	// See the comment in pin regarding ordering of the loads.
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// 尝试从其他的local shared列表尾部获取对象
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Try the victim cache. We do this after attempting to steal
	// from all primary caches because we want objects in the
	// victim cache to age out if at all possible.
    //尝试从上一轮幸存者缓存或者
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Mark the victim cache as empty for future gets don't bother
	// with it.
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}


//go:linkname sync_runtime_procPin sync.runtime_procPin
//go:nosplit
func sync_runtime_procPin() int {
	return procPin()
}

//go:linkname sync_runtime_procUnpin sync.runtime_procUnpin
//go:nosplit
func sync_runtime_procUnpin() {
	procUnpin()
}

//开始gc前调用
func poolCleanup() {
	// This function is called with the world stopped, at the beginning of a garbage collection.
	// It must not allocate and probably should not call any runtime functions.
	// Because the world is stopped, no pool user can be in a
	// pinned section (in effect, this has all Ps pinned).
	// Drop victim caches from all pools.
    //函数会在STW截断调用，在垃圾回收之间
    //因为在STW不用上锁
  	//清理上一轮的pool
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	//当前的移动到victim
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	// The pools with non-empty primary caches now have non-empty
	// victim caches and no pools have primary caches.
    
	oldPools, allPools = allPools, nil
}








//runtime
//go:nosplit
func procPin() int {
	_g_ := getg()
	mp := _g_.m

	mp.locks++
	return int(mp.p.ptr().id)
}

//go:nosplit
func procUnpin() {
	_g_ := getg()
	_g_.m.locks--
}
```



sync.Pool结构

<img src="..\images\sync-pool.png" style="zoom:80%;" />



pool.pin()流程

<img src="..\images\sync-pool-flow.png" style="zoom: 50%;" />