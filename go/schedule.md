### 	schedule

https://blog.csdn.net/u010853261/article/details/84790392

https://www.qcrao.com/2019/09/02/dive-into-go-scheduler/

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/

https://segmentfault.com/a/1190000016038785

https://www.cnblogs.com/flhs/tag/Go%E6%BA%90%E7%A0%81/

https://mp.weixin.qq.com/mp/homepage?__biz=MzU1OTg5NDkzOA==&hid=1&sn=8fc2b63f53559bc0cee292ce629c4788&scene=25#wechat_redirect

https://www.cnblogs.com/flhs/p/12682881.html



- **mstart**：主要是设置g0.stackguard0，g0.stackguard1。
- **mstart1**：调用save保存callerpc和callerpc到g0.sched。然后调用schedule开始调度循环。
- **schedule**：获得一个可执行的g。下面用gp代指。
- **execute**(gp *g, inheritTime bool)：绑定gp与当前m，状态改为_Grunning。
- **gogo**(buf *gobuf)：加载gp的上下文，跳转到buf.pc指向的函数。
- **执行buf.pc指向函数**。
- **goexit->goexit1**：调用mcall(goexit0)。
- **mcall**(fn func(*g))：保存当前g（也就是gp）的上下文；切换到g0及其栈，调用fn，参数为gp。
- **goexit0**(gp *g)：清零gp的属性，状态_Grunning改为_Gdead；dropg解绑m和gp；gfput放入队列；**schedule**重新调度。

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

1. 
2. G：代表一个goroutine实体，它有自己的栈内存，instruction pointer和一些相关信息(比如等待的channel等等)，是用于调度器调度的实体。
3. M：代表一个真正的内核OS线程，和POSIX里的thread差不多，属于真正执行指令的人。
4. P：代表M调度的上下文，可以把它看做一个局部的调度器，调度协程go代码在一个内核线程上跑。P是实现协程与内核线程的N:M映射关系的关键。P的上限是通过系统变量`runtime.GOMAXPROCS (numLogicalProcessors)`来控制的。golang启动时更新这个值，一般不建议修改这个值。P的数量也代表了golang代码执行的并发度，即有多少goroutine可以并行的运行。
5. schedt：runtime全局调度时使用的数据结构，这个实体其实只是一个壳，里面主要有M的全局idle队列，P的全局idle队列，一个全局的就绪的G队列以及一个runtime全局调度器级别的锁。当对M或P等做一些非局部调度器的操作时，一般需要先锁住全局调度器。

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



除了上图中可能触发调度的时间点，运行时还会在线程启动 [`runtime.mstart`](https://draveness.me/golang/tree/runtime.mstart) 和 Goroutine 执行结束 [`runtime.goexit0`](https://draveness.me/golang/tree/runtime.goexit0) 触发调度。我们在这里会重点介绍运行时触发调度的几个路径：

- 主动挂起 — [`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) -> [`runtime.park_m`](https://draveness.me/golang/tree/runtime.park_m)
- 系统调用 — [`runtime.exitsyscall`](https://draveness.me/golang/tree/runtime.exitsyscall) -> [`runtime.exitsyscall0`](https://draveness.me/golang/tree/runtime.exitsyscall0)
- 协作式调度 — [`runtime.Gosched`](https://draveness.me/golang/tree/runtime.Gosched) -> [`runtime.gosched_m`](https://draveness.me/golang/tree/runtime.gosched_m) -> [`runtime.goschedImpl`](https://draveness.me/golang/tree/runtime.goschedImpl)
- 系统监控 — [`runtime.sysmon`](https://draveness.me/golang/tree/runtime.sysmon) -> [`runtime.retake`](https://draveness.me/golang/tree/runtime.retake) -> [`runtime.preemptone`](https://draveness.me/golang/tree/runtime.preemptone)

我们在这里介绍的调度时间点不是将线程的运行权直接交给其他任务，而是通过调度器的 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 重新调度。



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

_Gidle	刚刚被分配并且还没有被初始化
_Grunnable	没有执行代码，没有栈的所有权，存储在运行队列中
_Grunning	可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P
_Gsyscall	正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上
_Gwaiting	由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上
_Gdead	没有被使用，没有执行代码，可能有分配的栈
_Gcopystack	栈正在被拷贝，没有执行代码，不在运行队列上
_Gpreempted	由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒
_Gscan	GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在

type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // 当前 Goroutine 的栈内存范围 [stack.lo, stack.hi)
	stackguard0 uintptr // 用于调度器抢占式调度
	stackguard1 uintptr // offset known to liblink

	_panic       *_panic // innermost panic - offset known to liblink
	_defer       *_defer // innermost defer
	m            *m      // 当前 Goroutine 占用的线程，可能为空
	sched        gobuf  //存储 Goroutine 的调度相关的数据 上下文
	syscallsp    uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc    uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stktopsp     uintptr        // expected sp at top of stack, to check in traceback
	param        unsafe.Pointer // passed parameter on wakeup
	atomicstatus uint32  //Goroutine 的状态
	stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
	goid         int64
	schedlink    guintptr
	waitsince    int64      // approx time when the g become blocked
	waitreason   waitReason // if status==Gwaiting

	preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt   抢占信号
	preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule  抢占时将状态修改成 `_Gpreempted`
	preemptShrink bool // shrink stack at synchronous safe point 在同步安全点收缩栈

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
	g0      *g     //  g0 是持有调度栈的 Goroutine，
	morebuf gobuf  // gobuf arg to morestack
	divmod  uint32 // div/mod denominator for arm - known to liblink

	// Fields not known to debuggers.
	procid        uint64       // for debuggers, but offset not hard-coded
	gsignal       *g           // signal-handling g
	goSigStack    gsignalStack // Go-allocated signal handling stack
	sigmask       sigset       // storage for saved signal mask
	tls           [6]uintptr   // thread-local storage (for x86 extern register)
	mstartfn      func()   // 表示M的起始函数。其实就是我们 go 语句携带的那个函数。
	curg          *g       // 是在当前线程上运行的用户 Goroutine
	caughtsig     guintptr // goroutine running during fatal signal
	p             puintptr //与m绑定的p如果为nil表示空闲
	nextp         puintptr // 用于暂存于当前M有潜在关联的P。 （预联）当M重新启动时，即用预联的这个P做关联啦
	oldp          puintptr // 执行系统调用之前使用线程的处理器 oldp
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
	//gc
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
	// 检查定时器
   checkTimers(pp, 0)

   var gp *g
   var inheritTime bool

   // Normal goroutines will check for need to wakeP in ready,
   // but GCworkers and tracereaders will not, so the check must
   // be done here instead.
   tryWakeP := false

   if gp == nil && gcBlackenEnabled != 0 {
      gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
      tryWakeP = tryWakeP || gp != nil
   }
   if gp == nil {
      // 每调度61次检查全局runnable队列确保公平
      // 否则两个goroutine完全占据本地的runqueue,彼此不断调度
      if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
         lock(&sched.lock)
         gp = globrunqget(_g_.m.p.ptr(), 1)
         unlock(&sched.lock)
      }
   }
   if gp == nil {
       // 从本地的队列找
      gp, inheritTime = runqget(_g_.m.p.ptr())
       // 我么可以看到gp != nil 即使m在空转
       // 如果checkTimers通过goready添加一个本地goroutine
   }
   if gp == nil {
       //查找要执行的可运行goroutine。
		//尝试从其他P窃取，从本地或全局队列获取g，轮询网络。
          //如果从本地运行队列和全局运行队列都没有找到需要运行的goroutine，
        //则调用findrunnable函数从其它工作线程的运行队列中偷取，如果偷取不到，则当前工作线程进入睡眠，
        //直到获取到需要运行的goroutine之后findrunnable函数才会返回。
      gp, inheritTime = findrunnable() // blocks until work is available
   }

   // This thread is going to run a goroutine and is not spinning anymore,
   // so if it was marked as spinning we need to reset it now and potentially
   // start a new spinning M.
   // 这个线程将要执行goroutine并且不再自旋
   // 因此我们将要重置标记
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
	  //当前运行的是runtime的代码，函数调用栈使用的是g0的栈空间
    //调用execte切换到gp的代码和栈空间去运行
   execute(gp, inheritTime)
}


// 创建一个新的siz大小的参数的g
// 放到等待队列中等待运行 编译器将go语句翻译为这个方法
// 这个调用的栈不太一样，他假设传递给fn的参数紧跟在fn地址后，因此，他们是逻辑上newproc的一部分，即使他们并没有出现在参数列表
// （因为他们的类型不同在不同的调用地方）

// 这一定是nosplit，因为变量在当前地址后面，栈复制将不会调整他们栈分裂也不会复制他们


调用gfget获取newg，如果为nil，调用malg分配一个，然后加入到全局变量allgs中。
从调用newproc的函数栈帧中copy参数到newg栈帧中。
设置newg.sched属性，调用gostartcallfn，将newg和函数关联。
更改状态为_Grunnable，存储到p.runq中（p.runq长度是256，满了会被拿出一些放在sched.runq中）。
概括讲就是：获取g->复制参数->设置调度属性->放入队列等调度。

func newproc(siz int32, fn *funcval) {
     //函数调用参数入栈顺序是从右向左，而且栈是从高地址向低地址增长的
    //注意：argp指向fn函数的第一个参数，而不是newproc函数的参数
    //参数fn在栈上的地址+8的位置存放的是fn函数的第一个参数
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	 // 获取一个g（m.g0） 初始化时是m0.g0
    gp := getg()
      //getcallerpc()返回一个地址，也就是调用newproc时由call指令压栈的函数返回地址，
    //初始化的时候，pc就是CALLruntime·newproc(SB)指令后面的POPQ AX这条指令的地址	
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
	 //因为已经切换到g0栈，所以无论什么场景都有 _g_ = g0，当然这个g0是指当前工作线程的m.g0
    //对于初始化，当前工作线程是主线程，所以这里的g0 = m0.g0
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
    //从p的本地找g,
    //如果没有从全局sched.gFree移动到本地的gFree
	newg := gfget(_p_)
	if newg == nil {
        //本地和全局都没有g创建2k大小的g
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)  //初始化g的状态为_Gdead
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
	if newg.stack.hi == 0 {
		throw("newproc1: newg missing stack")
	}

	if readgstatus(newg) != _Gdead {
		throw("newproc1: new g is not Gdead")
	}
	   //调整g的栈顶置针
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
        //拷贝参数
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
	    //把newg.sched结构体成员的所有成员设置为0
	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	newg.sched.sp = sp
	newg.stktopsp = sp
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
    //入口函数
	gostartcallfn(&newg.sched, fn)
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
    //设置newg的startpc为fn.fn，该成员主要用于函数调用栈的traceback和栈收缩
    //newg真正从哪里开始执行并不依赖于这个成员，而是sched.pc
	newg.startpc = fn.fn
	if _g_.m.curg != nil {
		newg.labels = _g_.m.curg.labels
	}
	if isSystemGoroutine(newg, false) {
		atomic.Xadd(&sched.ngsys, +1)
	}
    //状态变更为可运行
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

func gostartcallfn(gobuf *gobuf, fv *funcval) {
	var fn unsafe.Pointer
        // fn是真正指向函数的指针
	if fv != nil {
		fn = unsafe.Pointer(fv.fn)
	} else {
        //空func
		fn = unsafe.Pointer(funcPC(nilfunc))
	}
	gostartcall(gobuf, fn, unsafe.Pointer(fv))
}

调整newg的栈空间，把goexit函数的第二条指令的地址入栈，伪造成goexit函数调用了fn，从而使fn执行完成后执行ret指令时返回到goexit继续执行完成最后的清理工作；
重新设置newg.buf.pc 为需要执行的函数的地址，即fn，我们这个场景为runtime.main函数的地址。

func gostartcall(buf *gobuf, fn, ctxt unsafe.Pointer) {
	sp := buf.sp
	if sys.RegSize > sys.PtrSize {
		sp -= sys.PtrSize
		*(*uintptr)(unsafe.Pointer(sp)) = 0
	}
	sp -= sys.PtrSize	// 为返回地址预留空间
        // buf.pc 存储的是 funcPC(goexit) + sys.PCQuantum 
        // 将其存储到返回地址是为了伪造成 fn 是被 goexit 调用的，在 fn 执行完后返回 goexit执行，做一些清理工作。
	*(*uintptr)(unsafe.Pointer(sp)) = buf.pc
	buf.sp = sp		// 重新赋值
	buf.pc = uintptr(fn)	// 赋值为函数指针
	buf.ctxt = ctxt
}

// Tries to add one more P to execute G's.
// Called when a G is made runnable (newproc, ready).
func wakep() {
    //如果有空闲的p
	if atomic.Load(&sched.npidle) == 0 {
		return
	}
	// be conservative about spinning threads
	if atomic.Load(&sched.nmspinning) != 0 || !atomic.Cas(&sched.nmspinning, 0, 1) {
		return
	}
	startm(nil, true)
}

// Schedules some M to run the p (creates an M if necessary).
// If p==nil, tries to get an idle P, if no idle P's does nothing.
// May run with m.p==nil, so write barriers are not allowed.
// If spinning is set, the caller has incremented nmspinning and startm will
// either decrement nmspinning or set m.spinning in the newly started M.
//go:nowritebarrierrec
startm函数首先判断是否有空闲的p结构体对象，如果没有则直接返回，如果有则需要创建或唤醒一个工作线程出来与之绑定，从这里可以看出所谓的唤醒p，其实就是把空闲的p利用起来。

在确保有可以绑定的p对象之后，startm函数首先尝试从m的空闲队列中查找正处于休眠状态的工作线程，如果找到则通过notewakeup函数唤醒它，否则调用newm函数创建一个新的工作线程出来。
func startm(_p_ *p, spinning bool) {
	lock(&sched.lock)
    //没有指定p的话需要从p的空闲队列中获取一个p
	if _p_ == nil {
		_p_ = pidleget()
		if _p_ == nil {
			unlock(&sched.lock)
			if spinning {
				// The caller incremented nmspinning, but there are no idle Ps,
				// so it's okay to just undo the increment and give up.
                 //spinning为true表示进入这个函数之前已经对sched.nmspinning加了1，需要还原
				if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
					throw("startm: negative nmspinning")
				}
			}
            //没有空闲的p，直接返回
			return
		}
	}
    //从m空闲队列中获取正处于睡眠之中的工作线程，所有处于睡眠状态的m都在此队列中
	mp := mget()
     //没有处于睡眠状态的工作线程
	if mp == nil {
		// No M is available, we must drop sched.lock and call newm.
		// However, we already own a P to assign to the M.
		//
		// Once sched.lock is released, another G (e.g., in a syscall),
		// could find no idle P while checkdead finds a runnable G but
		// no running M's because this new M hasn't started yet, thus
		// throwing in an apparent deadlock.
		//
		// Avoid this situation by pre-allocating the ID for the new M,
		// thus marking it as 'running' before we drop sched.lock. This
		// new M will eventually run the scheduler to execute any
		// queued G's.
		id := mReserveID()
		unlock(&sched.lock)

		var fn func()
		if spinning {
			// The caller incremented nmspinning, so set m.spinning in the new M.
			fn = mspinning
		}
        //创建新的工作线程
        //创建m绑定p
		newm(fn, _p_, id)
		return
	}
	unlock(&sched.lock)
	if mp.spinning {
		throw("startm: m is spinning")
	}
	if mp.nextp != 0 {
		throw("startm: m has p")
	}
	if spinning && !runqempty(_p_) {
		throw("startm: p has runnable gs")
	}
	// The caller incremented nmspinning, so set m.spinning in the new M.
	mp.spinning = spinning
    //m绑定p
	mp.nextp.set(_p_)
    //唤醒
	notewakeup(&mp.park)
}


// Schedules gp to run on the current M.
// If inheritTime is true, gp inherits the remaining time in the
// current time slice. Otherwise, it starts a new time slice.
// Never returns.
//
// Write barriers are allowed because this is called immediately after
// acquiring a P in several places.
//
//go:yeswritebarrierrec
func execute(gp *g, inheritTime bool) {
	_g_ := getg() //g0

	// Assign gp.m before entering _Grunning so running Gs have an
	// M.
	_g_.m.curg = gp
	gp.m = _g_.m
    //设置待运行g的状态为_Grunning
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		_g_.m.p.ptr().schedtick++
	}

	// Check whether the profiler needs to be turned on or off.
	hz := sched.profilehz
	if _g_.m.profilehz != hz {
		setThreadCPUProfiler(hz)
	}

	 //gogo完成从g0到gp真正的切换
	gogo(&gp.sched)
}

# func gogo(buf *gobuf)
把gp.sched的成员恢复到CPU的寄存器完成状态以及栈的切换；
跳转到gp.sched.pc所指的指令地址（go func）处执行
# restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $16-8
    #buf = &gp.sched
    MOVQ  buf+0(FP), BX   # BX = buf
  
    #gobuf->g --> dx register
    MOVQ  gobuf_g(BX), DX  # DX = gp.sched.g
  
    #下面这行代码没有实质作用，检查gp.sched.g是否是nil，如果是nil进程会crash死掉
    MOVQ  0(DX), CX   # make sure g != nil
  
    get_tls(CX) 
  
    #把要运行的g的指针放入线程本地存储，这样后面的代码就可以通过线程本地存储
    #获取到当前正在执行的goroutine的g结构体对象，从而找到与之关联的m和p
    MOVQ  DX, g(CX)
  
    #把CPU的SP寄存器设置为sched.sp，完成了栈的切换
    MOVQ  gobuf_sp(BX), SP  # restore SP
  
    #下面三条同样是恢复调度上下文到CPU相关寄存器
    MOVQ  gobuf_ret(BX), AX
    MOVQ  gobuf_ctxt(BX), DX
    MOVQ  gobuf_bp(BX), BP
  
    #清空sched的值，因为我们已把相关值放入CPU对应的寄存器了，不再需要，这样做可以少gc的工作量
    MOVQ  $0, gobuf_sp(BX)  # clear to help garbage collector
    MOVQ  $0, gobuf_ret(BX)
    MOVQ  $0, gobuf_ctxt(BX)
    MOVQ  $0, gobuf_bp(BX)
  
    #把sched.pc值放入BX寄存器
    MOVQ  gobuf_pc(BX), BX
  
    #JMP把BX寄存器的包含的地址值放入CPU的IP寄存器，于是，CPU跳转到该地址继续执行指令，
    JMP BX


// Finishes execution of the current goroutine.
func goexit1() {
	mcall(goexit0)
}

// mcall从用户g切换到g0并调用fn
// mcall保存当前g的pc和sp在g->sched等待将来恢复，由fn来安排稍后的执行，通常是通过记录
// 数据结构中的g，导致稍后调用ready（g）。
// mcall稍后返回原始的g，当g呗重新调度，fn必须没有返回，通常通过调用schedule结束，让m运行其他groutine

首先从当前运行的g(我们这个场景是g2)切换到g0，这一步包括保存当前g的调度信息，把g0设置到tls中，修改CPU的rsp寄存器使其指向g0的栈
以当前运行的g为参数调用fn函数(此处为goexit0)。

从mcall的功能我们可以看出，mcall做的事情跟gogo函数完全相反，gogo函数实现了从g0切换到某个goroutine去运行，而mcall实现了从某个goroutine切换到g0来运行，因此，mcall和gogo的代码非常相似，然而mcall和gogo在做切换时有个重要的区别：gogo函数在从g0切换到其它goroutine时首先切换了栈，然后通过跳转指令从runtime代码切换到了用户goroutine的代码，而mcall函数在从其它goroutine切换回g0时只切换了栈，并未使用跳转指令跳转到runtime代码去执行。为什么会有这个差别呢？原因在于在从g0切换到其它goroutine之前执行的是runtime的代码而且使用的是g0栈，所以切换时需要首先切换栈然后再从runtime代码跳转某个goroutine的代码去执行（切换栈和跳转指令不能颠倒，因为跳转之后执行的就是用户的goroutine代码了，没有机会切换栈了），然而从某个goroutine切换回g0时，goroutine使用的是call指令来调用mcall函数，mcall函数本身就是runtime的代码，所以call指令其实已经完成了从goroutine代码到runtime代码的跳转，因此mcall函数自身的代码就不需要再跳转了，只需要把栈切换到g0栈即可。



func mcall(fn func(*g))

# mcall的参数是一个指向funcval对象的指针
TEXT runtime·mcall(SB), NOSPLIT, $0-8
    #取出参数的值放入DI寄存器，它是funcval对象的指针，此场景中fn.fn是goexit0的地址
    MOVQ  fn+0(FP), DI

    get_tls(CX)
    MOVQ  g(CX), AX # AX = g，本场景g 是 g2

    #mcall返回地址放入BX
    MOVQ  0(SP), BX# caller's PC

    #保存g2的调度信息，因为我们要从当前正在运行的g2切换到g0
    MOVQ  BX, (g_sched+gobuf_pc)(AX)   #g.sched.pc = BX，保存g2的rip
    LEAQ  fn+0(FP), BX # caller's SP  
    MOVQ  BX, (g_sched+gobuf_sp)(AX)  #g.sched.sp = BX，保存g2的rsp
    MOVQ  AX, (g_sched+gobuf_g)(AX)   #g.sched.g = g
    MOVQ  BP, (g_sched+gobuf_bp)(AX)  #g.sched.bp = BP，保存g2的rbp

    # switch to m->g0 & its stack, call fn
    #下面三条指令主要目的是找到g0的指针
    MOVQ  g(CX), BX         #BX = g
    MOVQ  g_m(BX), BX    #BX = g.m
    MOVQ  m_g0(BX), SI   #SI = g.m.g0

    #此刻，SI = g0， AX = g，所以这里在判断g 是否是 g0，如果g == g0则一定是哪里代码写错了
    CMPQ  SI, AX# if g == m->g0 call badmcall
    JNE  3(PC)
    MOVQ  $runtime·badmcall(SB), AX
    JMP  AX

    #把g0的地址设置到线程本地存储之中
    MOVQ  SI, g(CX)

    #恢复g0的栈顶指针到CPU的rsp积存，这一条指令完成了栈的切换，从g的栈切换到了g0的栈
    MOVQ  (g_sched+gobuf_sp)(SI), SP# rsp = g0->sched.sp

    #AX = g
    PUSHQ  AX   #fn的参数g入栈
    MOVQ  DI, DX   #DI是结构体funcval实例对象的指针，它的第一个成员才是goexit0的地址
    MOVQ  0(DI), DI   #读取第一个成员到DI寄存器
    CALL  DI   #调用goexit0(g)
    POPQ  AX
    MOVQ  $runtime·badmcall2(SB), AX
    JMP  AX
    RET

把g的状态从_Grunning变更为_Gdead；
然后把g的一些字段清空成0值；
调用dropg函数解除g和m之间的关系，其实就是设置g->m = nil, m->currg = nil；
把g放入p的freeg队列缓存起来供下次创建g时快速获取而不用从内存分配。freeg就是g的一个对象池；
调用schedule函数再次进行调度

//调用链路
schedule()->execute()->gogo()->g2()->goexit()->goexit1()->mcall()->goexit0()->schedule()

func goexit0(gp *g) {
	_g_ := getg()

	casgstatus(gp, _Grunning, _Gdead)
	if isSystemGoroutine(gp, false) {
		atomic.Xadd(&sched.ngsys, -1)
	}
	gp.m = nil
	locked := gp.lockedm != 0
	gp.lockedm = 0
	_g_.m.lockedg = 0
	gp.preemptStop = false
	gp.paniconfault = false
	gp._defer = nil // should be true already but just in case.
	gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
	gp.writebuf = nil
	gp.waitreason = 0
	gp.param = nil
	gp.labels = nil
	gp.timer = nil

	if gcBlackenEnabled != 0 && gp.gcAssistBytes > 0 {
		// Flush assist credit to the global pool. This gives
		// better information to pacing if the application is
		// rapidly creating an exiting goroutines.
		scanCredit := int64(gcController.assistWorkPerByte * float64(gp.gcAssistBytes))
		atomic.Xaddint64(&gcController.bgScanCredit, scanCredit)
		gp.gcAssistBytes = 0
	}
	//解除绑定
    
	dropg()

	if GOARCH == "wasm" { // no threads yet on wasm
		gfput(_g_.m.p.ptr(), gp)
		schedule() // never returns
	}

	if _g_.m.lockedInt != 0 {
		print("invalid m->lockedInt = ", _g_.m.lockedInt, "\n")
		throw("internal lockOSThread error")
	}
    //当前p的gFree>=64 移动一半到全局sched.gFree
	gfput(_g_.m.p.ptr(), gp)
	if locked {
		// The goroutine may have locked this thread because
		// it put it in an unusual kernel state. Kill it
		// rather than returning it to the thread pool.

		// Return to mstart, which will release the P and exit
		// the thread.
		if GOOS != "plan9" { // See golang.org/issue/22227.
			gogo(&_g_.m.g0.sched)
		} else {
			// Clear lockedExt on plan9 since we may end up re-using
			// this thread.
			_g_.m.lockedExt = 0
		}
	}
	schedule()
}

//由于直接进行系统调用会阻塞当前的线程，所以只有可以立刻返回的系统调用才可能会被设置成 RawSyscall 类型，例如：SYS_EPOLL_CREATE、SYS_EPOLL_WAIT（超时时间为 0）、SYS_TIME 等。

// Standard syscall entry used by the go syscall library and normal cgo calls.
//
// This is exported via linkname to assembly in the syscall package.
//
//go:nosplit
//go:linkname entersyscall
func entersyscall() {
	reentersyscall(getcallerpc(), getcallersp())
}
// The goroutine g is about to enter a system call.
// Record that it's not using the cpu anymore.
// This is called only from the go syscall library and cgocall,
// not from the low-level system calls used by the runtime.
//
// Entersyscall cannot split the stack: the gosave must
// make g->sched refer to the caller's stack segment, because
// entersyscall is going to return immediately after.
//
// Nothing entersyscall calls can split the stack either.
// We cannot safely move the stack during an active call to syscall,
// because we do not know which of the uintptr arguments are
// really pointers (back into the stack).
// In practice, this means that we make the fast path run through
// entersyscall doing no-split things, and the slow path has to use systemstack
// to run bigger things on the system stack.
//
// reentersyscall is the entry point used by cgo callbacks, where explicitly
// saved SP and PC are restored. This is needed when exitsyscall will be called
// from a function further up in the call stack than the parent, as g->syscallsp
// must always point to a valid stack frame. entersyscall below is the normal
// entry point for syscalls, which obtains the SP and PC from the caller.
//
// Syscall tracing:
// At the start of a syscall we emit traceGoSysCall to capture the stack trace.
// If the syscall does not block, that is it, we do not emit any other events.
// If the syscall blocks (that is, P is retaken), retaker emits traceGoSysBlock;
// when syscall returns we emit traceGoSysExit and when the goroutine starts running
// (potentially instantly, if exitsyscallfast returns true) we emit traceGoStart.
// To ensure that traceGoSysExit is emitted strictly after traceGoSysBlock,
// we remember current value of syscalltick in m (_g_.m.syscalltick = _g_.m.p.ptr().syscalltick),
// whoever emits traceGoSysBlock increments p.syscalltick afterwards;
// and we wait for the increment before emitting traceGoSysExit.
// Note that the increment is done even if tracing is not enabled,
// because tracing can be enabled in the middle of syscall. We don't want the wait to hang.
//
//go:nosplit
func reentersyscall(pc, sp uintptr) {
	_g_ := getg()

	// Disable preemption because during this function g is in Gsyscall status,
	// but can have inconsistent g->sched, do not let GC observe it.
	_g_.m.locks++

	// Entersyscall must not call any function that might split/grow the stack.
	// (See details in comment above.)
	// Catch calls that might, by replacing the stack guard with something that
	// will trip any stack check and leaving a flag to tell newstack to die.
	_g_.stackguard0 = stackPreempt
	_g_.throwsplit = true

	// Leave SP around for GC and traceback.
	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
	casgstatus(_g_, _Grunning, _Gsyscall)
	if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
		systemstack(func() {
			print("entersyscall inconsistent ", hex(_g_.syscallsp), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
			throw("entersyscall")
		})
	}

	if trace.enabled {
		systemstack(traceGoSysCall)
		// systemstack itself clobbers g.sched.{pc,sp} and we might
		// need them later when the G is genuinely blocked in a
		// syscall
		save(pc, sp)
	}

	if atomic.Load(&sched.sysmonwait) != 0 {
		systemstack(entersyscall_sysmon)
		save(pc, sp)
	}

	if _g_.m.p.ptr().runSafePointFn != 0 {
		// runSafePointFn may stack split if run on this stack
		systemstack(runSafePointFn)
		save(pc, sp)
	}

	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick
	_g_.sysblocktraced = true
	pp := _g_.m.p.ptr()
	pp.m = 0
	_g_.m.oldp.set(pp)
	_g_.m.p = 0
	atomic.Store(&pp.status, _Psyscall)
	if sched.gcwaiting != 0 {
		systemstack(entersyscall_gcwait)
		save(pc, sp)
	}

	_g_.m.locks--
}

我们在设计原理中介绍过了 Go 语言基于协作式和信号的两种抢占式调度，这里主要介绍其中的协作式调度。runtime.Gosched 函数会主动让出处理器，允许其他 Goroutine 运行。该函数无法挂起 Goroutine，调度器会在可能会将当前 Goroutine 调度到其他线程上：
// Gosched yields the processor, allowing other goroutines to run. It does not
// suspend the current goroutine, so execution resumes automatically.
func Gosched() {
	checkTimeouts()
	mcall(gosched_m)
}


// Create a new m. It will start off with a call to fn, or else the scheduler.
// fn needs to be static and not a heap allocated closure.
// May run with m.p==nil, so write barriers are not allowed.
//
// id is optional pre-allocated m ID. Omit by passing -1.
// sysmon不需要p
//go:nowritebarrierrec
func newm(fn func(), _p_ *p, id int64) {
    // 虚拟的m实际的os线程要newm1中申请
	mp := allocm(_p_, fn, id)
	mp.nextp.set(_p_)
	mp.sigmask = initSigmask
	if gp := getg(); gp != nil && gp.m != nil && (gp.m.lockedExt != 0 || gp.m.incgo) && GOOS != "plan9" {
		// We're on a locked M or a thread that may have been
		// started by C. The kernel state of this thread may
		// be strange (the user may have locked it for that
		// purpose). We don't want to clone that into another
		// thread. Instead, ask a known-good thread to create
		// the thread for us.
		//
		// This is disabled on Plan 9. See golang.org/issue/22227.
		//
		// TODO: This may be unnecessary on Windows, which
		// doesn't model thread creation off fork.
		lock(&newmHandoff.lock)
		if newmHandoff.haveTemplateThread == 0 {
			throw("on a locked thread with no template thread")
		}
		mp.schedlink = newmHandoff.newm
		newmHandoff.newm.set(mp)
		if newmHandoff.waiting {
			newmHandoff.waiting = false
			notewakeup(&newmHandoff.wake)
		}
		unlock(&newmHandoff.lock)
		return
	}
	newm1(mp)
}
func newm1(mp *m) {
	execLock.rlock() // Prevent process clone.
    //系统os
	newosproc(mp)
	execLock.runlock()
}

// Allocate a new m unassociated with any thread.
// Can use p for allocation context if needed.
// fn is recorded as the new m's m.mstartfn.
// id is optional pre-allocated m ID. Omit by passing -1.
//
// This function is allowed to have write barriers even if the caller
// isn't because it borrows _p_.
//
//go:yeswritebarrierrec
func allocm(_p_ *p, fn func(), id int64) *m {
	_g_ := getg()
	acquirem() // disable GC because it can be called from sysmon
	if _g_.m.p == 0 {
		acquirep(_p_) // temporarily borrow p for mallocs in this function
	}

	// Release the free M list. We need to do this somewhere and
	// this may free up a stack we can use.
	if sched.freem != nil {
		lock(&sched.lock)
		var newList *m
		for freem := sched.freem; freem != nil; {
			if freem.freeWait != 0 {
				next := freem.freelink
				freem.freelink = newList
				newList = freem
				freem = next
				continue
			}
			stackfree(freem.g0.stack)
			freem = freem.freelink
		}
		sched.freem = newList
		unlock(&sched.lock)
	}

	mp := new(m)
	mp.mstartfn = fn
	mcommoninit(mp, id)

	// In case of cgo or Solaris or illumos or Darwin, pthread_create will make us a stack.
	// Windows and Plan 9 will layout sched stack on OS stack.
	if iscgo || GOOS == "solaris" || GOOS == "illumos" || GOOS == "windows" || GOOS == "plan9" || GOOS == "darwin" {
		mp.g0 = malg(-1)
	} else {
		mp.g0 = malg(8192 * sys.StackGuardMultiplier)
	}
	mp.g0.m = mp

	if _p_ == _g_.m.p.ptr() {
		releasep()
	}
	releasem(_g_.m)

	return mp
}
```







### 调度时机

第一步，从全局运行队列中寻找goroutine。为了保证调度的公平性，每个工作线程每经过61次调度就需要优先尝试从全局运行队列中找出一个goroutine来运行，这样才能保证位于全局运行队列中的goroutine得到调度的机会。全局运行队列是所有工作线程都可以访问的，所以在访问它之前需要加锁。

第二步，从工作线程本地运行队列中寻找goroutine。如果不需要或不能从全局运行队列中获取到goroutine则从本地运行队列中获取。

第三步，从其它工作线程的运行队列中偷取goroutine。如果上一步也没有找到需要运行的goroutine，则调用findrunnable从其他工作线程的运行队列中偷取goroutine，findrunnable函数在偷取之前会再次尝试从全局运行队列和当前线程的本地运行队列中查找需要运行的goroutine。



```go
// Try get a batch of G's from the global runnable queue.
// Sched must be locked.
globrunqget函数首先会根据全局运行队列中goroutine的数量，函数参数max以及_p_的本地队列的容量计算出到底应该拿多少个goroutine，然后把第一个g结构体对象通过返回值的方式返回给调用函数，其它的则通过runqput函数放入当前工作线程的本地运行队列。这段代码值得一提的是，计算应该从全局运行队列中拿走多少个goroutine时根据p的数量（gomaxprocs）做了负载均衡。

func globrunqget(_p_ *p, max int32) *g {
	if sched.runqsize == 0 {
		return nil
	}
	//根据p的数量平分全局运行队列中的goroutines
	n := sched.runqsize/gomaxprocs + 1
	if n > sched.runqsize {
        //上面计算n的方法可能导致n大于全局运行队列中的goroutine数量
		n = sched.runqsize
	}
	if max > 0 && n > max {
        //最多取max个goroutine
        // max 可能是0
        // schedule中max=1
		n = max
	}
	if n > int32(len(_p_.runq))/2 {
        //最多只能取本地队列容量的一半
		n = int32(len(_p_.runq)) / 2
	}

	sched.runqsize -= n
	//直接通过函数返回gp，其它的goroutines通过runqput放入本地运行队列
	gp := sched.runq.pop()
	n--
	for ; n > 0; n-- {
        //从全局运行队列中取出一个goroutine
		gp1 := sched.runq.pop()
        //放入本地运行队列
		runqput(_p_, gp1, false)
	}
	return gp
}


// Get g from local runnable queue.
// If inheritTime is true, gp should inherit the remaining time in the
// current time slice. Otherwise, it should start a new time slice.
// Executed only by the owner P.
从代码上来看，工作线程的本地运行队列其实分为两个部分，一部分是由p的runq、runqhead和runqtail这三个成员组成的一个无锁循环队列，该队列最多可包含256个goroutine；另一部分是p的runnext成员，它是一个指向g结构体对象的指针，它最多只包含一个goroutine。

从本地运行队列中寻找goroutine是通过runqget函数完成的，寻找时，代码首先查看runnext成员是否为空，如果不为空则返回runnext所指的goroutine，并把runnext成员清零，如果runnext为空，则继续从循环队列中查找goroutine。


这里首先需要注意的是不管是从runnext还是从循环队列中拿取goroutine都使用了cas操作，这里的cas操作是必需的，因为可能有其他工作线程此时此刻也正在访问这两个成员，从这里偷取可运行的goroutine。
func runqget(_p_ *p) (gp *g, inheritTime bool) {
	// If there's a runnext, it's the next G to run.
	for {
         //查看runnext成员是否为空，不为空则返回该goroutine
		next := _p_.runnext
		if next == 0 {
			break
		}
        //置为空
		if _p_.runnext.cas(next, 0) {
			return next.ptr(), true
		}
	}
	 //从循环队列中获取goroutine
	for {
		h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
		t := _p_.runqtail
        //头==尾
		if t == h {
			return nil, false
		}
        //256取模
		gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
		if atomic.CasRel(&_p_.runqhead, h, h+1) { // cas-release, commits consume
			return gp, false
		}
	}
}


// runqput tries to put g on the local runnable queue.
// If next is false, runqput adds g to the tail of the runnable queue.
// If next is true, runqput puts g in the _p_.runnext slot.
// If the run queue is full, runnext puts g on the global queue.
// Executed only by the owner P.
runtime.runqput 会将 Goroutine 放到运行队列上，这既可能是全局的运行队列，也可能是处理器本地的运行队列：
当 next 为 true 时，将 Goroutine 设置到处理器的 runnext 作为下一个处理器执行的任务；
当 next 为 false 并且本地运行队列还有剩余空间时，将 Goroutine 加入处理器持有的本地运行队列；
当处理器的本地运行队列已经没有剩余空间时就会把本地队列中的一部分 Goroutine 和待加入的 Goroutine 通过 runtime.runqputslow 添加到调度器持有的全局运行队列上
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}
	 //把gp放在_p_.runnext成员里，
     //runnext成员中的goroutine会被优先调度起来运行
	if next {
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
            //有其它线程在操作runnext成员，需要重试
			goto retryNext
		}
         //原本runnext为nil，所以没任何事情可做了，直接返回
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
        //原本存放在runnext的gp需要放入runq的尾部
		gp = oldnext.ptr()
	}
 //可能有其它线程正在并发修改runqhead成员，所以需要跟其它线程同步
retry:
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
     //队列还没有满，可以放入
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
    //放到全局
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}

// Put g and a batch of work from local runnable queue on global queue.
// Executed only by the owner P.

runqputslow函数首先使用链表把从_p_的本地队列中取出的一半连同gp一起串联起来，然后在加锁成功之后通过globrunqputbatch函数把该链表链入全局运行队列（全局运行队列是使用链表实现的）。值的一提的是runqputslow函数并没有一开始就把全局运行队列锁住，而是等所有的准备工作做完之后才锁住全局运行队列，这是并发编程加锁的基本原则，需要尽量减小锁的粒度，降低锁冲突的概率。

func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
     //gp加上_p_本地队列的一半
	var batch [len(_p_.runq)/2 + 1]*g

	// First, grab a batch from local queue.
	n := t - h
	n = n / 2
	if n != uint32(len(_p_.runq)/2) {
		throw("runqputslow: queue is not full")
	}
	for i := uint32(0); i < n; i++ {
        //取出p本地队列的一半
		batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))].ptr()
	}
	if !atomic.CasRel(&_p_.runqhead, h, h+n) { // cas-release, commits consume
		return false
	}
	batch[n] = gp

	if randomizeScheduler {
		for i := uint32(1); i <= n; i++ {
			j := fastrandn(i + 1)
			batch[i], batch[j] = batch[j], batch[i]
		}
	}
	
	// Link the goroutines.
    //全局运行队列是一个链表，这里首先把所有需要放入全局运行队列的g链接起来，
    //减少后面对全局链表的锁住时间，从而降低锁冲突
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}
	var q gQueue
	q.head.set(batch[0])
	q.tail.set(batch[n])

	// Now put the batch on global queue.
	lock(&sched.lock)
	globrunqputbatch(&q, int32(n+1))
	unlock(&sched.lock)
	return true
}
```







第一点，**工作线程M的自旋状态(spinning)**。**工作线程在从其它工作线程的本地运行队列中盗取goroutine时的状态称为自旋状态**。从上面代码可以看到，当前M在去其它p的运行队列盗取goroutine之前把spinning标志设置成了true，同时增加处于自旋状态的M的数量，而盗取结束之后则把spinning标志还原为false，同时减少处于自旋状态的M的数量，从后面的分析我们可以看到，当有空闲P又有goroutine需要运行的时候，这个处于自旋状态的M的数量决定了是否需要唤醒或者创建新的工作线程。



第二点，**盗取算法**。盗取过程用了两个嵌套for循环。内层循环实现了盗取逻辑，从代码可以看出盗取的实质就是遍历allp中的所有p，查看其运行队列是否有goroutine，如果有，则取其一半到当前工作线程的运行队列，然后从findrunnable返回，如果没有则继续遍历下一个p。**但这里为了保证公平性，遍历allp时并不是固定的从allp[0]即第一个p开始，而是从随机位置上的p开始，而且遍历的顺序也随机化了，并不是现在访问了第i个p下一次就访问第i+1个p，而是使用了一种伪随机的方式遍历allp中的每个p，防止每次遍历时使用同样的顺序访问allp中的元素**。下面是这个算法的伪代码：







```go
// Finds a runnable goroutine to execute.
// Tries to steal from other P's, get g from local or global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
   _g_ := getg()

   // The conditions here and in handoffp must agree: if
   // findrunnable would return a G to run, handoffp must start
   // an M.

top:
   _p_ := _g_.m.p.ptr()
   if sched.gcwaiting != 0 {
      gcstopm()
      goto top
   }
   if _p_.runSafePointFn != 0 {
      runSafePointFn()
   }
	
   now, pollUntil, _ := checkTimers(_p_, 0)

   if fingwait && fingwake {
      if gp := wakefing(); gp != nil {
         ready(gp, 0, true)
      }
   }
   if *cgo_yield != nil {
      asmcgocall(*cgo_yield, nil)
   }

  //再次看一下本地运行队列是否有需要运行的goroutine
   if gp, inheritTime := runqget(_p_); gp != nil {
      return gp, inheritTime
   }

  //再看看全局运行队列是否有需要运行的goroutine
   if sched.runqsize != 0 {
      lock(&sched.lock)
      gp := globrunqget(_p_, 0)
      unlock(&sched.lock)
      if gp != nil {
         return gp, false
      }
   }

   // Poll network.
   // This netpoll is only an optimization before we resort to stealing.
   // We can safely skip it if there are no waiters or a thread is blocked
   // in netpoll already. If there is any kind of logical race with that
   // blocked thread (e.g. it has already returned from netpoll, but does
   // not set lastpoll yet), this thread will do blocking netpoll below
   // anyway.
   if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
      if list := netpoll(0); !list.empty() { // non-blocking
         gp := list.pop()
         injectglist(&list)
         casgstatus(gp, _Gwaiting, _Grunnable)
         if trace.enabled {
            traceGoUnpark(gp, 0)
         }
         return gp, false
      }
   }

   //如果除了当前工作线程还在运行外，其它工作线程已经处于休眠中，那么也就不用去偷了，肯定没有
   procs := uint32(gomaxprocs)
   ranTimer := false
   // If number of spinning M's >= number of busy P's, block.
   // This is necessary to prevent excessive CPU consumption
   // when GOMAXPROCS>>1 but the program parallelism is low.
    // 这个判断主要是为了防止因为寻找可运行的goroutine而消耗太多的CPU。
    // 因为已经有足够多的工作线程正在寻找可运行的goroutine，让他们去找就好了，自己偷个懒去睡觉
   if !_g_.m.spinning && 2*atomic.Load(&sched.nmspinning) >= procs-atomic.Load(&sched.npidle) {
      goto stop
   }
   if !_g_.m.spinning {
       //设置m的状态为spinning
      _g_.m.spinning = true
        //处于spinning状态的m数量加一
      atomic.Xadd(&sched.nmspinning, 1)
   }
    //从其它p的本地运行队列盗取goroutine
   for i := 0; i < 4; i++ {
      for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
         if sched.gcwaiting != 0 {
            goto top
         }
         stealRunNextG := i > 2 // first look for ready queues with more than 1 g
         p2 := allp[enum.position()]
         if _p_ == p2 {
            continue
         }
         if gp := runqsteal(_p_, p2, stealRunNextG); gp != nil {
            return gp, false
         }

         // Consider stealing timers from p2.
         // This call to checkTimers is the only place where
         // we hold a lock on a different P's timers.
         // Lock contention can be a problem here, so
         // initially avoid grabbing the lock if p2 is running
         // and is not marked for preemption. If p2 is running
         // and not being preempted we assume it will handle its
         // own timers.
         // If we're still looking for work after checking all
         // the P's, then go ahead and steal from an active P.
         if i > 2 || (i > 1 && shouldStealTimers(p2)) {
            tnow, w, ran := checkTimers(p2, now)
            now = tnow
            if w != 0 && (pollUntil == 0 || w < pollUntil) {
               pollUntil = w
            }
            if ran {
               // Running the timers may have
               // made an arbitrary number of G's
               // ready and added them to this P's
               // local run queue. That invalidates
               // the assumption of runqsteal
               // that is always has room to add
               // stolen G's. So check now if there
               // is a local G to run.
               if gp, inheritTime := runqget(_p_); gp != nil {
                  return gp, inheritTime
               }
               ranTimer = true
            }
         }
      }
   }
   if ranTimer {
      // Running a timer may have made some goroutine ready.
      goto top
   }

stop:

   // We have nothing to do. If we're in the GC mark phase, can
   // safely scan and blacken objects, and have work to do, run
   // idle-time marking rather than give up the P.
   if gcBlackenEnabled != 0 && _p_.gcBgMarkWorker != 0 && gcMarkWorkAvailable(_p_) {
      _p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
      gp := _p_.gcBgMarkWorker.ptr()
      casgstatus(gp, _Gwaiting, _Grunnable)
      if trace.enabled {
         traceGoUnpark(gp, 0)
      }
      return gp, false
   }

   delta := int64(-1)
   if pollUntil != 0 {
      // checkTimers ensures that polluntil > now.
      delta = pollUntil - now
   }

   // wasm only:
   // If a callback returned and no other goroutine is awake,
   // then wake event handler goroutine which pauses execution
   // until a callback was triggered.
   gp, otherReady := beforeIdle(delta)
   if gp != nil {
      casgstatus(gp, _Gwaiting, _Grunnable)
      if trace.enabled {
         traceGoUnpark(gp, 0)
      }
      return gp, false
   }
   if otherReady {
      goto top
   }

   // Before we drop our P, make a snapshot of the allp slice,
   // which can change underfoot once we no longer block
   // safe-points. We don't need to snapshot the contents because
   // everything up to cap(allp) is immutable.
   allpSnapshot := allp

   // return P and block
   lock(&sched.lock)
   if sched.gcwaiting != 0 || _p_.runSafePointFn != 0 {
      unlock(&sched.lock)
      goto top
   }
    
   if sched.runqsize != 0 {
      gp := globrunqget(_p_, 0)
      unlock(&sched.lock)
      return gp, false
   }
     // 当前工作线程解除与p之间的绑定，准备去休眠
   if releasep() != _p_ {
      throw("findrunnable: wrong p")
   }
    //把p放入空闲队列
   pidleput(_p_)
   unlock(&sched.lock)

   // Delicate dance: thread transitions from spinning to non-spinning state,
   // potentially concurrently with submission of new goroutines. We must
   // drop nmspinning first and then check all per-P queues again (with
   // #StoreLoad memory barrier in between). If we do it the other way around,
   // another thread can submit a goroutine after we've checked all run queues
   // but before we drop nmspinning; as the result nobody will unpark a thread
   // to run the goroutine.
   // If we discover new work below, we need to restore m.spinning as a signal
   // for resetspinning to unpark a new worker thread (because there can be more
   // than one starving goroutine). However, if after discovering new work
   // we also observe no idle Ps, it is OK to just park the current thread:
   // the system is fully loaded so no spinning threads are required.
   // Also see "Worker thread parking/unparking" comment at the top of the file.
   wasSpinning := _g_.m.spinning
   if _g_.m.spinning {
     //m即将睡眠，状态不再是spinning
      _g_.m.spinning = false
      if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
         throw("findrunnable: negative nmspinning")
      }
   }

   // check all runqueues once again
     // 休眠之前再看一下是否有工作要做
   for _, _p_ := range allpSnapshot {
      if !runqempty(_p_) {
         lock(&sched.lock)
         _p_ = pidleget()
         unlock(&sched.lock)
         if _p_ != nil {
            acquirep(_p_)
            if wasSpinning {
               _g_.m.spinning = true
               atomic.Xadd(&sched.nmspinning, 1)
            }
            goto top
         }
         break
      }
   }

   // Check for idle-priority GC work again.
   if gcBlackenEnabled != 0 && gcMarkWorkAvailable(nil) {
      lock(&sched.lock)
      _p_ = pidleget()
      if _p_ != nil && _p_.gcBgMarkWorker == 0 {
         pidleput(_p_)
         _p_ = nil
      }
      unlock(&sched.lock)
      if _p_ != nil {
         acquirep(_p_)
         if wasSpinning {
            _g_.m.spinning = true
            atomic.Xadd(&sched.nmspinning, 1)
         }
         // Go back to idle GC check.
         goto stop
      }
   }

   // poll network
   if netpollinited() && (atomic.Load(&netpollWaiters) > 0 || pollUntil != 0) && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
      atomic.Store64(&sched.pollUntil, uint64(pollUntil))
      if _g_.m.p != 0 {
         throw("findrunnable: netpoll with p")
      }
      if _g_.m.spinning {
         throw("findrunnable: netpoll with spinning")
      }
      if faketime != 0 {
         // When using fake time, just poll.
         delta = 0
      }
      list := netpoll(delta) // block until new work is available
      atomic.Store64(&sched.pollUntil, 0)
      atomic.Store64(&sched.lastpoll, uint64(nanotime()))
      if faketime != 0 && list.empty() {
         // Using fake time and nothing is ready; stop M.
         // When all M's stop, checkdead will call timejump.
         stopm()
         goto top
      }
      lock(&sched.lock)
      _p_ = pidleget()
      unlock(&sched.lock)
      if _p_ == nil {
         injectglist(&list)
      } else {
         acquirep(_p_)
         if !list.empty() {
            gp := list.pop()
            injectglist(&list)
            casgstatus(gp, _Gwaiting, _Grunnable)
            if trace.enabled {
               traceGoUnpark(gp, 0)
            }
            return gp, false
         }
         if wasSpinning {
            _g_.m.spinning = true
            atomic.Xadd(&sched.nmspinning, 1)
         }
         goto top
      }
   } else if pollUntil != 0 && netpollinited() {
      pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
      if pollerPollUntil == 0 || pollerPollUntil > pollUntil {
         netpollBreak()
      }
   }
    //休眠
   stopm()
   goto top
}

// Stops execution of the current m until new work is available.
// Returns with acquired P.
stopm的核心是调用mput把m结构体对象放入sched的midle空闲队列，然后通过notesleep(&m.park)函数让自己进入睡眠状态。

note是go runtime实现的一次性睡眠和唤醒机制，一个线程可以通过调用notesleep(*note)进入睡眠状态，而另外一个线程则可以通过notewakeup(*note)把其唤醒。note的底层实现机制跟操作系统相关，不同系统使用不同的机制，比如linux下使用的futex系统调用，而mac下则是使用的pthread_cond_t条件变量，note对这些底层机制做了一个抽象和封装，这种封装给扩展性带来了很大的好处，比如当睡眠和唤醒功能需要支持新平台时，只需要在note层增加对特定平台的支持即可，不需要修改上层的任何代码。

func stopm() {
	_g_ := getg()

	if _g_.m.locks != 0 {
		throw("stopm holding locks")
	}
	if _g_.m.p != 0 {
		throw("stopm holding p")
	}
	if _g_.m.spinning {
		throw("stopm spinning")
	}

	lock(&sched.lock)
	mput(_g_.m)
	unlock(&sched.lock)
	notesleep(&_g_.m.park)
   
     //唤醒后 需要再次绑定一个p，然后返回到findrunnable函数继续重新寻找可运行的goroutine，一旦找到可运行的goroutine就会返回到schedule函数，并把找到的goroutine调度起来运行，如何把goroutine调度起来运行的代码我们已经分析过了。现在继续看notesleep函数。
	noteclear(&_g_.m.park)
	acquirep(_g_.m.nextp.ptr())
	_g_.m.nextp = 0
}

func notesleep(n *note) {
	gp := getg()
	if gp != gp.m.g0 {
		throw("notesleep not on g0")
	}
	semacreate(gp.m)
	if !atomic.Casuintptr(&n.key, 0, uintptr(unsafe.Pointer(gp.m))) {
		// Must be locked (got wakeup).
		if n.key != locked {
			throw("notesleep - waitm out of sync")
		}
		return
	}
	// Queued. Sleep.
	gp.m.blocked = true
	if *cgo_yield == nil {
		semasleep(-1)
	} else {
		// Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
		const ns = 10e6
		for atomic.Loaduintptr(&n.key) == 0 {
			semasleep(ns)
			asmcgocall(*cgo_yield, nil)
		}
	}
	gp.m.blocked = false
}

```







```go
// func systemstack(fn func())
TEXT runtime·systemstack(SB), NOSPLIT, $0-8
   MOVQ   fn+0(FP), DI   // DI = fn
   get_tls(CX)       // m tls保存的g
   MOVQ   g(CX), AX  // AX = g
   MOVQ   g_m(AX), BX    // BX = m

   CMPQ   AX, m_gsignal(BX)
   JEQ    noswitch

   MOVQ   m_g0(BX), DX   // DX = g0  
   CMPQ   AX, DX         // 当前的g和g0做比较 如果当前g不是g0切换
   JEQ    noswitch

   CMPQ   AX, m_curg(BX)
   JNE    bad

   // switch stacks
   // save our state in g->sched. Pretend to
   // be systemstack_switch if the G stack is scanned.
   MOVQ   $runtime·systemstack_switch(SB), SI
   MOVQ   SI, (g_sched+gobuf_pc)(AX)
   MOVQ   SP, (g_sched+gobuf_sp)(AX)
   MOVQ   AX, (g_sched+gobuf_g)(AX)
   MOVQ   BP, (g_sched+gobuf_bp)(AX)

   // switch to g0
   MOVQ   DX, g(CX)
   MOVQ   (g_sched+gobuf_sp)(DX), BX
   // make it look like mstart called systemstack on g0, to stop traceback
   SUBQ   $8, BX
   MOVQ   $runtime·mstart(SB), DX
   MOVQ   DX, 0(BX)
   MOVQ   BX, SP

   // call target function
   MOVQ   DI, DX
   MOVQ   0(DI), DI
   CALL   DI

   // switch back to g
   get_tls(CX)
   MOVQ   g(CX), AX
   MOVQ   g_m(AX), BX
   MOVQ   m_curg(BX), AX
   MOVQ   AX, g(CX)
   MOVQ   (g_sched+gobuf_sp)(AX), SP
   MOVQ   $0, (g_sched+gobuf_sp)(AX)
   RET

noswitch:
   // already on m stack; tail call the function
   // Using a tail call here cleans up tracebacks since we won't stop
   // at an intermediate systemstack.
   MOVQ   DI, DX
   MOVQ   0(DI), DI
   JMP    DI

bad:
   // Bad: g is not gsignal, not g0, not curg. What is it?
   MOVQ   $runtime·badsystemstack(SB), AX
   CALL   AX
   INT    $3
```

