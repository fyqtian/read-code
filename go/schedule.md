### schedule

https://blog.csdn.net/u010853261/article/details/84790392

https://www.qcrao.com/2019/09/02/dive-into-go-scheduler/

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/

https://segmentfault.com/a/1190000016038785

| 中文名                | 源码名称             | 作用域     | 简要说明                                 |
| --------------------- | -------------------- | ---------- | ---------------------------------------- |
| 全局M列表             | runtime.allm         | 运行时系统 | 存放所有M                                |
| 全局P列表             | runtime.allp         | 运行时系统 | 存放所有P                                |
| 全局G列表             | runtime.allg         | 运行时系统 | 存放所有G                                |
|                       |                      |            |                                          |
| 调度器中的空闲M列表   | runtime.schedt.midle | 调度器     | 存放空闲M，链表结构                      |
| 调度器中的空闲P列表   | runtime.schedt.pidle | 调度器     | 存放空闲P，链表结构                      |
| 调度器中的可运行G队列 | runtime.schedt.runq  | 调度器     | 存放可运行G，链表结构                    |
| 调度器中的自由G列表   | runtime.schedt.gfree | 调度器     | 存放自由G， 链表结构                     |
|                       |                      |            |                                          |
| P中的可运行G队列      | runq                 | 本地P      | 存放当前P中的可运行G，环形队列，数组实现 |
| P中的自由G列表        | gfree                | 本地P      | 存放当前P中的自由G，链表结构             |

1. G：代表一个goroutine实体，它有自己的栈内存，instruction pointer和一些相关信息(比如等待的channel等等)，是用于调度器调度的实体。
2. M：代表一个真正的内核OS线程，和POSIX里的thread差不多，属于真正执行指令的人。
3. P：代表M调度的上下文，可以把它看做一个局部的调度器，调度协程go代码在一个内核线程上跑。P是实现协程与内核线程的N:M映射关系的关键。P的上限是通过系统变量`runtime.GOMAXPROCS (numLogicalProcessors)`来控制的。golang启动时更新这个值，一般不建议修改这个值。P的数量也代表了golang代码执行的并发度，即有多少goroutine可以并行的运行。
4. schedt：runtime全局调度时使用的数据结构，这个实体其实只是一个壳，里面主要有M的全局idle队列，P的全局idle队列，一个全局的就绪的G队列以及一个runtime全局调度器级别的锁。当对M或P等做一些非局部调度器的操作时，一般需要先锁住全局调度器。

<img src="..\images\20181204172621290.png" alt="20181204172621290" style="zoom:67%;" />



1. 我们通过 go func()来创建一个goroutine；

2. 有两个存储goroutine的队列，一个是局部调度器P的`local queue`、一个是全局调度器数据模型`schedt`的global queue。新创建的goroutine会先保存在local queue，如果local queue已经满了就会保存在全局的global queue；

3. goroutine只能运行在M中，一个M必须持有一个P，M与P是1：1的关系。M会从P的local queue弹出一个Runable状态的goroutine来执行，如果P的local queue为空，就会执行work stealing；

4. 一个M调度goroutine执行的过程是一个loop

5. 当M执行某一个goroutine时候如果发生了syscall或则其余阻塞操作，M会阻塞，如果当前有一些G在执行，runtime会把这个线程M从P中摘除(detach)，然后再创建一个新的操作系统的线程(如果有空闲的线程可用就复用空闲线程)来服务于这个P；

6. 当M系统调用结束时候，这个goroutine会尝试获取一个空闲的P执行，并放入到这个P的local queue。如果获取不到P，那么这个线程M会park它自己(休眠)， 加入到空闲线程中，然后这个goroutine会被放入schedt的global queue。

   

   Go运行时会在下面的goroutine被阻塞的情况下运行另外一个goroutine：

   - syscall
   - network input
   - channel operations
   - primitives in the sync package

   

<img src="..\images\20181205151403287.png" alt="20181205151403287" style="zoom: 67%;" />

从上图我们可以看到，**除了Pdead状态以外的其余状态，在runtime进行GC的时候，P都会被指定成Pgcstop。在GC结束后状态不会回复到GC前的状态，而是都统一直接转到了Pidle 【这意味着，他们都需要被重新调度】。**

【注意】除了Pgcstop 状态的P，其他状态的P都会在调用runtime.GOMAXPROCS 函数减少P数目时，被认为是多余的P而状态转为Pdead，这时候其带的可运行G的队列中的G都会被转移到调度器的可运行G队列中，它的自由G队列 【gfree】也是一样被移到调度器的自由列表【runtime.sched.gfree】中。

【注意】每个P中都有一个可运行G队列及自由G队列。自由G队列包含了很多已经完成的G，随着被运行完成的G的积攒到一定程度后，runtime会把其中的部分G转移到全局调度器的自由G队列 【runtime.sched.gfree】中。

【注意】当我们每次用 go关键字启用一个G的时候，首先都是尽可能复用已经执行完的G。具体过程如下：运行时系统都会先从P的自由G队列获取一个G来封装我们提供的函数 (go 关键字后面的函数) ，如果发现P中的自由G过少时，会从调度器的自由G队列中移一些G过来，只有连调度器的自由G列表都弹尽粮绝的时候，才会去创建新的G。



<img src="..\images\2018120516114851.png" alt="2018120516114851" style="zoom:67%;" />



```go
Go语言的编译器会把我们编写的goroutine编译为runtime的函数调用，并把go语句中的函数以及其参数传递给runtime的函数中。

runtime在接到这样一个调用后，会先检查一下go函数及其参数的合法性，紧接着会试图从局部调度器P的自由G队列中(或者全局调度器的自由G队列)中获取一个可用的自由G （P中有讲述了），如果没有则新创建一个G。类似M和P，G在运行时系统中也有全局的G列表【runtime.allg】，那些新建的G会先放到这个全局的G列表中，其列表的作用也是集中放置了当前运行时系统中给所有的G的指针。在用自由G封装go的函数时，运行时系统都会对这个G重新做一次初始化。

初始化：包含了被关联的go关键字后的函数及当前G的状态机G的ID等等。在G被初始化完成后就会被放置到当前本地的P的可运行队列中。只要时机成熟，调度器会立即尽心这个G的调度运行。

G的状态机会比较复杂一点，大致上和内核线程的状态机有一点类似，但是状态机流转有一些区别。G的各种状态如下：

Gidle：G被创建但还未完全被初始化。
Grunnable：当前G为可运行的，正在等待被运行。
Grunning：当前G正在被运行。
Gsyscall：当前G正在被系统调用
Gwaiting：当前G正在因某个原因而等待
Gdead：当前G完成了运行

初始化完的G是处于Grunnable的状态，一个G真正在M中运行时是处于Grunning的状态，G的状态机流转图如下图所示：

上图有一步是等待的事件到来，那么G在运行过程中，是否等待某个事件以及等待什么样的事件？完全由起封装的go关键字后的函数决定。（如：等待chan中的值、涉及网络I/O、time.Timer、time.Sleep等等事件）

G退出系统调用的过程非常复杂：runtime先会尝试获取空闲局部调度器P并直接运行当前G，如果没有就会把当前G转成Grunnable状态并放置入全局调度器的global queue。

最后，已经是Gdead状态的G是可以被重新初始化并使用的(从自由G队列取出来重新初始化使用)。而对比进入Pdead状态的P等待的命运只有被销毁。处于Gdead的G会被放置到本地P或者调度器的自由G列表中。

type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	_panic       *_panic // innermost panic - offset known to liblink
	_defer       *_defer // innermost defer
	m            *m      // current m; offset known to arm liblink
	sched        gobuf
	syscallsp    uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc    uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stktopsp     uintptr        // expected sp at top of stack, to check in traceback
	param        unsafe.Pointer // passed parameter on wakeup
	atomicstatus uint32
	stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
	goid         int64
	schedlink    guintptr
	waitsince    int64      // approx time when the g become blocked
	waitreason   waitReason // if status==Gwaiting

	preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
	preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
	preemptShrink bool // shrink stack at synchronous safe point

	// asyncSafePoint is set if g is stopped at an asynchronous
	// safe point. This means there are frames on the stack
	// without precise pointer information.
	asyncSafePoint bool

	paniconfault bool // panic (instead of crash) on unexpected fault address
	gcscandone   bool // g has scanned stack; protected by _Gscan bit in status
	throwsplit   bool // must not split stack
	// activeStackChans indicates that there are unlocked channels
	// pointing into this goroutine's stack. If true, stack
	// copying needs to acquire channel locks to protect these
	// areas of the stack.
	activeStackChans bool

	raceignore     int8     // ignore race detection events
	sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
	sysexitticks   int64    // cputicks when syscall has returned (for tracing)
	traceseq       uint64   // trace event sequencer
	tracelastp     puintptr // last P emitted an event for this goroutine
	lockedm        muintptr
	sig            uint32
	writebuf       []byte
	sigcode0       uintptr
	sigcode1       uintptr
	sigpc          uintptr
	gopc           uintptr         // pc of go statement that created this goroutine
	ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
	startpc        uintptr         // pc of goroutine function
	racectx        uintptr
	waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
	cgoCtxt        []uintptr      // cgo traceback context
	labels         unsafe.Pointer // profiler labels
	timer          *timer         // cached timer for time.Sleep
	selectDone     uint32         // are we participating in a select and did someone win the race?

	// Per-G GC state

	// gcAssistBytes is this G's GC assist credit in terms of
	// bytes allocated. If this is positive, then the G has credit
	// to allocate gcAssistBytes bytes without assisting. If this
	// is negative, then the G must correct this by performing
	// scan work. We track this in bytes to make it fast to update
	// and check for debt in the malloc hot path. The assist ratio
	// determines how this corresponds to scan work debt.
	gcAssistBytes int64
}

自旋中(spinning): M正在从运行队列获取G, 这时候M会拥有一个P；
执行go代码中: M正在执行go代码, 这时候M会拥有一个P；
执行原生代码中: M正在执行原生代码或者阻塞的syscall, 这时M并不拥有P；
休眠中: M发现无待运行的G时会进入休眠，并添加到空闲M链表中, 这时M并不拥有P。

type m struct {
    /*
        1.  所有调用栈的Goroutine,这是一个比较特殊的Goroutine。
        2.  普通的Goroutine栈是在Heap分配的可增长的stack,而g0的stack是M对应的线程栈。
        3.  所有与调度相关的代码,都会先切换到g0的栈再执行。
    */
	g0      *g     // 线程栈
	morebuf gobuf  // gobuf arg to morestack
	divmod  uint32 // div/mod denominator for arm - known to liblink

	// Fields not known to debuggers.
	procid        uint64       // for debuggers, but offset not hard-coded
	gsignal       *g           // signal-handling g
	goSigStack    gsignalStack // Go-allocated signal handling stack
	sigmask       sigset       // storage for saved signal mask
	tls           [6]uintptr   // thread-local storage (for x86 extern register)
	mstartfn      func()   // 表示M的起始函数。其实就是我们 go 语句携带的那个函数。
	curg          *g       // M中当前运行的goroutine
	caughtsig     guintptr // goroutine running during fatal signal
	p             puintptr //与m绑定的p如果为nil表示空闲
	nextp         puintptr // 用于暂存于当前M有潜在关联的P。 （预联）当M重新启动时，即用预联的这个P做关联啦
	oldp          puintptr // the p that was attached before executing a syscall
	id            int64
	mallocing     int32
	throwing      int32
	preemptoff    string // 是否抢占 if != "", keep curg running on this m
	locks         int32
	dying         int32
	profilehz     int32
	spinning      bool // m is out of work and is actively looking for work
	blocked       bool // m is blocked on a note
	newSigstack   bool // minit on C thread called sigaltstack
	printlock     int8
	incgo         bool   // m is executing a cgo call
	freeWait      uint32 // if == 0, safe to free g0 and delete m (atomic)
	fastrand      [2]uint32
	needextram    bool
	traceback     uint8
	ncgocall      uint64      // number of cgo calls in total
	ncgo          int32       // number of cgo calls currently in progress
	cgoCallersUse uint32      // if non-zero, cgoCallers in use temporarily
	cgoCallers    *cgoCallers // cgo traceback if crashing in cgo call
	park          note
	alllink       *m // on allm
	schedlink     muintptr
	lockedg       guintptr  // 表示与当前M锁定的那个G。运行时系统会把 一个M 和一个G锁定，一旦锁定就只能双方相互作用，不接受第三者。
	createstack   [32]uintptr // stack that created this thread.
	lockedExt     uint32      // tracking for external LockOSThread
	lockedInt     uint32      // tracking for internal lockOSThread
	nextwaitm     muintptr    // next m waiting for lock
	waitunlockf   func(*g, unsafe.Pointer) bool
	waitlock      unsafe.Pointer
	waittraceev   byte
	waittraceskip int
	startingtrace bool
	syscalltick   uint32
	freelink      *m // on sched.freem

	// these are here because they are too large to be on the stack
	// of low-level NOSPLIT functions.
	libcall   libcall
	libcallpc uintptr // for cpu profiler
	libcallsp uintptr
	libcallg  guintptr
	syscall   libcall // stores syscall parameters on windows

	vdsoSP uintptr // SP for traceback while in VDSO call (0 if not in call)
	vdsoPC uintptr // PC for traceback while in VDSO call

	// preemptGen counts the number of completed preemption
	// signals. This is used to detect when a preemption is
	// requested, but fails. Accessed atomically.
	preemptGen uint32

	// Whether this is a pending preemption signal on this M.
	// Accessed atomically.
	signalPending uint32

	dlogPerM

	mOS

	// Up to 10 locks held by this m, maintained by the lock ranking code.
	locksHeldLen int
	locksHeld    [10]heldLockInfo
}

通过runtime.GOMAXPROCS函数我们可以改变单个Go程序可以拥有P的最大数量，如果不做设置会有一个默认值。
每一个P都必须关联一个M才能使其中的G得以运行。

【注意】：runtime会将M与关联的P分离开来。但是如果该P的runqueue中还有未运行的G，那么runtime就会找到一个空的M（在调度器的空闲队列中的M） 或者创建一个空的M，并与该P关联起来（为了运行G而做准备）。

runtime.GOMAXPROCS只能够设置P的数量，并不会影响到M（内核线程）数量，所以runtime.GOMAXPROCS不是控制线程数，只能影响局部调度器P的数量

runtime初始化时会确认P的最大数量，之后会根据这个最大值初始化全局P列表【runtime.allp】。类似全局M列表，【runtime.allp】包含了runtime创建的所有P。随后，runtime会把调度器的可运行G队列【runtime.schedt.runq】中的所有G均匀的放入全局的P列表中的各个P的可执行G队列 local queue中。到这里为止，runtime需要用到的所有P都准备就绪了

类似M的空闲列表，调度器也存在一个空闲P的列表【runtime.shcedt.pidle】，当一个P不再与任何M关联的时候，runtime会把该P放入这个列表，而一个空闲的P关联了某个M之后会被从【runtime.shcedt.pidle】中取出来。【注意：一个P加入了空闲列表，其G的可运行local queue也不一定为空】。

Pidel：当前P未和任何M关联
Prunning：当前P已经和某个M关联，M在执行某个G
Psyscall：当前P中的被运行的那个G正在进行系统调用
Pgcstop：runtime正在进行GC（runtime会在gc时试图把全局P列表中的P都处于此种状态）
Pdead：当前P已经不再被使用（在调用runtime.GOMAXPROCS减少P的数量时，多余的P就处于此状态）

type p struct {
	id          int32
	status      uint32 	  //当前状态 // one of pidle/prunning/...
	link        puintptr  //p连接
	schedtick   uint32     // incremented on every scheduler call
	syscalltick uint32     // incremented on every system call
	sysmontick  sysmontick // last tick observed by sysmon
	m           muintptr   // 反向关联到p（空闲时为nil）
	mcache      *mcache    // 
	pcache      pageCache
	raceprocctx uintptr

	deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
	deferpoolbuf [5][32]*_defer

	// Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
	goidcache    uint64
	goidcacheend uint64

	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	runnext guintptr

	// Available G's (status == Gdead)
	gFree struct {
		gList
		n int32
	}

	sudogcache []*sudog
	sudogbuf   [128]*sudog

	// Cache of mspan objects from the heap.
	mspancache struct {
		// We need an explicit length here because this field is used
		// in allocation codepaths where write barriers are not allowed,
		// and eliminating the write barrier/keeping it eliminated from
		// slice updates is tricky, moreso than just managing the length
		// ourselves.
		len int
		buf [128]*mspan
	}

	tracebuf traceBufPtr

	// traceSweep indicates the sweep events should be traced.
	// This is used to defer the sweep start event until a span
	// has actually been swept.
	traceSweep bool
	// traceSwept and traceReclaimed track the number of bytes
	// swept and reclaimed by sweeping in the current sweep loop.
	traceSwept, traceReclaimed uintptr

	palloc persistentAlloc // per-P to avoid mutex

	_ uint32 // Alignment for atomic fields below

	// The when field of the first entry on the timer heap.
	// This is updated using atomic functions.
	// This is 0 if the timer heap is empty.
	timer0When uint64

	// Per-P GC state
	gcAssistTime         int64    // Nanoseconds in assistAlloc
	gcFractionalMarkTime int64    // Nanoseconds in fractional mark worker (atomic)
	gcBgMarkWorker       guintptr // (atomic)
	gcMarkWorkerMode     gcMarkWorkerMode

	// gcMarkWorkerStartTime is the nanotime() at which this mark
	// worker started.
	gcMarkWorkerStartTime int64

	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork

	// wbBuf is this P's GC write barrier buffer.
	//
	// TODO: Consider caching this in the running G.
	wbBuf wbBuf

	runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

	// Lock for timers. We normally access the timers while running
	// on this P, but the scheduler can also do it from a different P.
	timersLock mutex

	// Actions to take at some time. This is used to implement the
	// standard library's time package.
	// Must hold timersLock to access.
	timers []*timer

	// Number of timers in P's heap.
	// Modified using atomic instructions.
	numTimers uint32

	// Number of timerModifiedEarlier timers on P's heap.
	// This should only be modified while holding timersLock,
	// or while the timer status is in a transient state
	// such as timerModifying.
	adjustTimers uint32

	// Number of timerDeleted timers in P's heap.
	// Modified using atomic instructions.
	deletedTimers uint32

	// Race context used while executing timer functions.
	timerRaceCtx uintptr

	// preempt is set to indicate that this P should be enter the
	// scheduler ASAP (regardless of what G is running on it).
	preempt bool

	pad cpu.CacheLinePad
}


// 一轮调度找到可运行的goroutine并执行
func schedule() {
    //m当前g
   _g_ := getg()
	
   if _g_.m.locks != 0 {
      throw("schedule: holding locks")
   }
	//有
   if _g_.m.lockedg != 0 {
      stoplockedm()
      execute(_g_.m.lockedg.ptr(), false) // Never returns.
   }

   // We should not schedule away from a g that is executing a cgo call,
   // since the cgo call is using the m's g0 stack.
   if _g_.m.incgo {
      throw("schedule: in cgo")
   }

top:
   pp := _g_.m.p.ptr()
   pp.preempt = false

   if sched.gcwaiting != 0 {
      gcstopm()
      goto top
   }
   if pp.runSafePointFn != 0 {
      runSafePointFn()
   }

   // Sanity check: if we are spinning, the run queue should be empty.
   // Check this before calling checkTimers, as that might call
   // goready to put a ready goroutine on the local run queue.
   if _g_.m.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
      throw("schedule: spinning with local work")
   }

   checkTimers(pp, 0)

   var gp *g
   var inheritTime bool

   // Normal goroutines will check for need to wakeP in ready,
   // but GCworkers and tracereaders will not, so the check must
   // be done here instead.
   tryWakeP := false
   if trace.enabled || trace.shutdown {
      gp = traceReader()
      if gp != nil {
         casgstatus(gp, _Gwaiting, _Grunnable)
         traceGoUnpark(gp, 0)
         tryWakeP = true
      }
   }
   if gp == nil && gcBlackenEnabled != 0 {
      gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
      tryWakeP = tryWakeP || gp != nil
   }
   if gp == nil {
      // Check the global runnable queue once in a while to ensure fairness.
      // Otherwise two goroutines can completely occupy the local runqueue
      // by constantly respawning each other.
      if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
         lock(&sched.lock)
         gp = globrunqget(_g_.m.p.ptr(), 1)
         unlock(&sched.lock)
      }
   }
   if gp == nil {
      gp, inheritTime = runqget(_g_.m.p.ptr())
      // We can see gp != nil here even if the M is spinning,
      // if checkTimers added a local goroutine via goready.
   }
   if gp == nil {
      gp, inheritTime = findrunnable() // blocks until work is available
   }

   // This thread is going to run a goroutine and is not spinning anymore,
   // so if it was marked as spinning we need to reset it now and potentially
   // start a new spinning M.
   if _g_.m.spinning {
      resetspinning()
   }

   if sched.disable.user && !schedEnabled(gp) {
      // Scheduling of this goroutine is disabled. Put it on
      // the list of pending runnable goroutines for when we
      // re-enable user scheduling and look again.
      lock(&sched.lock)
      if schedEnabled(gp) {
         // Something re-enabled scheduling while we
         // were acquiring the lock.
         unlock(&sched.lock)
      } else {
         sched.disable.runnable.pushBack(gp)
         sched.disable.n++
         unlock(&sched.lock)
         goto top
      }
   }

   // If about to schedule a not-normal goroutine (a GCworker or tracereader),
   // wake a P if there is one.
   if tryWakeP {
      wakep()
   }
   if gp.lockedm != 0 {
      // Hands off own p to the locked m,
      // then blocks waiting for a new p.
      startlockedm(gp)
      goto top
   }

   execute(gp, inheritTime)
}


// 创建一个新的siz大小的参数的g
// 放到等待队列中等待运行 编译器将go语句翻译为这个方法
// 这个调用的栈不太一样，他假设传递给fn的参数紧跟在fn地址后，因此，他们是逻辑上newproc的一部分，即使他们并没有出现在参数列表
// （因为他们的类型不同在不同的调用地方）

// 这一定是nosplit，因为变量在当前地址后面，栈复制将不会调整他们栈分裂也不会复制他们

func newproc(siz int32, fn *funcval) {
    //参数
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	//当前g
    gp := getg()
	pc := getcallerpc()
	systemstack(func() {
		//创建新的g
        newg := newproc1(fn, argp, siz, gp, pc)
		_p_ := getg().m.p.ptr()
        // 放到当前p 如果p满了放到全局
		runqput(_p_, newg, true)
		// 如果主线程已经执行 判断是否有idel的p，有就创建m绑定p
		if mainStarted {
			wakep()
		}
	})
}


// 创建一个g状态在_Grunnable，入口在fn，narg字节个参数在地址argp,callerpc是是调用者的PC，调用者负责把g加入到调度
// 必须运行在系统栈，因为他是无法拆分栈
//go:systemstack
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
	_g_ := getg()

	if fn == nil {
		_g_.m.throwing = -1 // do not dump full stacks
		throw("go of nil func value")
	}
	acquirem() // disable preemption because it can be holding p in a local var
	siz := narg
	siz = (siz + 7) &^ 7

	// We could allocate a larger initial stack if necessary.
	// Not worth it: this is almost always an error.
	// 4*sizeof(uintreg): extra space added below
	// sizeof(uintreg): caller's LR (arm) or return address (x86, in gostartcall).
	if siz >= _StackMin-4*sys.RegSize-sys.RegSize {
		throw("newproc: function arguments too large for new goroutine")
	}

	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_)
	if newg == nil {
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
	if newg.stack.hi == 0 {
		throw("newproc1: newg missing stack")
	}

	if readgstatus(newg) != _Gdead {
		throw("newproc1: new g is not Gdead")
	}

	totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
	totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
	sp := newg.stack.hi - totalSize
	spArg := sp
	if usesLR {
		// caller's LR
		*(*uintptr)(unsafe.Pointer(sp)) = 0
		prepGoExitFrame(sp)
		spArg += sys.MinFrameSize
	}
	if narg > 0 {
		memmove(unsafe.Pointer(spArg), argp, uintptr(narg))
		// This is a stack-to-stack copy. If write barriers
		// are enabled and the source stack is grey (the
		// destination is always black), then perform a
		// barrier copy. We do this *after* the memmove
		// because the destination stack may have garbage on
		// it.
		if writeBarrier.needed && !_g_.m.curg.gcscandone {
			f := findfunc(fn.fn)
			stkmap := (*stackmap)(funcdata(f, _FUNCDATA_ArgsPointerMaps))
			if stkmap.nbit > 0 {
				// We're in the prologue, so it's always stack map index 0.
				bv := stackmapdata(stkmap, 0)
				bulkBarrierBitmap(spArg, spArg, uintptr(bv.n)*sys.PtrSize, 0, bv.bytedata)
			}
		}
	}

	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	newg.sched.sp = sp
	newg.stktopsp = sp
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	gostartcallfn(&newg.sched, fn)
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
	newg.startpc = fn.fn
	if _g_.m.curg != nil {
		newg.labels = _g_.m.curg.labels
	}
	if isSystemGoroutine(newg, false) {
		atomic.Xadd(&sched.ngsys, +1)
	}
	casgstatus(newg, _Gdead, _Grunnable)

	if _p_.goidcache == _p_.goidcacheend {
		// Sched.goidgen is the last allocated id,
		// this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
		// At startup sched.goidgen=0, so main goroutine receives goid=1.
		_p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
		_p_.goidcache -= _GoidCacheBatch - 1
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
	}
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
	if raceenabled {
		newg.racectx = racegostart(callerpc)
	}
	if trace.enabled {
		traceGoCreate(newg, newg.startpc)
	}
	releasem(_g_.m)

	return newg
}

```