### bootstrap

https://blog.csdn.net/QQ1130141391/article/details/96197570

https://mp.weixin.qq.com/mp/homepage?__biz=MzU1OTg5NDkzOA==&hid=1&sn=8fc2b63f53559bc0cee292ce629c4788&scene=25#wechat_redirect

https://www.cnblogs.com/flhs/p/12677335.html



gdb execfile

找到entrypoint

(gdb) info file
Symbols from "/opt/gocode/test/chan/test".
Local exec file:
        `/opt/gocode/test/chan/test', file type elf64-x86-64.
        Entry point: 0x464680
        0x0000000000401000 - 0x0000000000499174 is .text
        0x000000000049a000 - 0x00000000004de066 is .rodata
        0x00000000004de240 - 0x00000000004de974 is .typelink
        0x00000000004de978 - 0x00000000004de9c8 is .itablink
        0x00000000004de9c8 - 0x00000000004de9c8 is .gosymtab
        0x00000000004de9e0 - 0x000000000053e5db is .gopclntab
        0x000000000053f000 - 0x000000000053f020 is .go.buildinfo
        0x000000000053f020 - 0x000000000054d4c0 is .noptrdata
        0x000000000054d4c0 - 0x0000000000554930 is .data
        0x0000000000554940 - 0x0000000000584850 is .bss
        0x0000000000584860 - 0x00000000005870a8 is .noptrbss
        0x0000000000400f9c - 0x0000000000401000 is .note.go.buildid

b *0x464680

(gdb) b *0x464680
Breakpoint 1 at 0x464680: file /usr/local/go/src/runtime/rt0_linux_amd64.s, line 8.
(gdb)                                                                                                                                                                                                    



 打印变量p 'runtime.g0'

```go
src/runtime/rt0_linux_amd64.s, line 8
TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB)

src/runtime/asm_amd64.s, line 14
// _rt0_amd64 is common startup code for most amd64 systems when using
// internal linking. This is the entry point for the program from the
// kernel for an ordinary -buildmode=exe program. The stack holds the
// number of arguments and the C-style argv.
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB)

TEXT runtime·rt0_go(SB) src/runtime/asm_amd64.s:89

```

```go
// _rt0_amd64_lib is common startup code for most amd64 systems when
// using -buildmode=c-archive or -buildmode=c-shared. The linker will
// arrange to invoke this function as a global constructor (for
// c-archive) or when the shared library is loaded (for c-shared).
// We expect argc and argv to be passed in the usual C ABI registers
// DI and SI.
TEXT _rt0_amd64_lib(SB),NOSPLIT,$0x50
   // Align stack per ELF ABI requirements.
   MOVQ   SP, AX
   ANDQ   $~15, SP
   // Save C ABI callee-saved registers, as caller may need them.
   MOVQ   BX, 0x10(SP)
   MOVQ   BP, 0x18(SP)
   MOVQ   R12, 0x20(SP)
   MOVQ   R13, 0x28(SP)
   MOVQ   R14, 0x30(SP)
   MOVQ   R15, 0x38(SP)
   MOVQ   AX, 0x40(SP)

   MOVQ   DI, _rt0_amd64_lib_argc<>(SB)
   MOVQ   SI, _rt0_amd64_lib_argv<>(SB)

   // Synchronous initialization.
   CALL   runtime·libpreinit(SB)

   // Create a new thread to finish Go runtime initialization.
   MOVQ   _cgo_sys_thread_create(SB), AX
   TESTQ  AX, AX
   JZ nocgo
   MOVQ   $_rt0_amd64_lib_go(SB), DI
   MOVQ   $0, SI
   CALL   AX
   JMP    restore

nocgo:
   MOVQ   $0x800000, 0(SP)      // stacksize
   MOVQ   $_rt0_amd64_lib_go(SB), AX
   MOVQ   AX, 8(SP)        // fn
   CALL   runtime·newosproc0(SB)

restore:
   MOVQ   0x10(SP), BX
   MOVQ   0x18(SP), BP
   MOVQ   0x20(SP), R12
   MOVQ   0x28(SP), R13
   MOVQ   0x30(SP), R14
   MOVQ   0x38(SP), R15
   MOVQ   0x40(SP), SP
   RET

// _rt0_amd64_lib_go initializes the Go runtime.
// This is started in a separate thread by _rt0_amd64_lib.
TEXT _rt0_amd64_lib_go(SB),NOSPLIT,$0
   MOVQ   _rt0_amd64_lib_argc<>(SB), DI
   MOVQ   _rt0_amd64_lib_argv<>(SB), SI
   JMP    runtime·rt0_go(SB)

DATA _rt0_amd64_lib_argc<>(SB)/8, $0
GLOBL _rt0_amd64_lib_argc<>(SB),NOPTR, $8
DATA _rt0_amd64_lib_argv<>(SB)/8, $0
GLOBL _rt0_amd64_lib_argv<>(SB),NOPTR, $8

TEXT runtime·rt0_go(SB),NOSPLIT,$0
   // copy arguments forward on an even stack
   MOVQ   DI, AX    // argc
   MOVQ   SI, BX    // argv
   SUBQ   $(4*8+7), SP      // 2args 2auto
   ANDQ   $~15, SP
   MOVQ   AX, 16(SP)
   MOVQ   BX, 24(SP)

   // create istack out of the given (operating system) stack.
   // _cgo_init may update stackguard.
   MOVQ   $runtime·g0(SB), DI
   LEAQ   (-64*1024+104)(SP), BX
   MOVQ   BX, g_stackguard0(DI)
   MOVQ   BX, g_stackguard1(DI)
   MOVQ   BX, (g_stack+stack_lo)(DI)
   MOVQ   SP, (g_stack+stack_hi)(DI)

   // find out information about the processor we're on
   MOVL   $0, AX
   CPUID
   MOVL   AX, SI
   CMPL   AX, $0
   JE nocpuinfo

   // Figure out how to serialize RDTSC.
   // On Intel processors LFENCE is enough. AMD requires MFENCE.
   // Don't know about the rest, so let's do MFENCE.
   CMPL   BX, $0x756E6547  // "Genu"
   JNE    notintel
   CMPL   DX, $0x49656E69  // "ineI"
   JNE    notintel
   CMPL   CX, $0x6C65746E  // "ntel"
   JNE    notintel
   MOVB   $1, runtime·isIntel(SB)
   MOVB   $1, runtime·lfenceBeforeRdtsc(SB)
notintel:

   // Load EAX=1 cpuid flags
   MOVL   $1, AX
   CPUID
   MOVL   AX, runtime·processorVersionInfo(SB)

nocpuinfo:
   // if there is an _cgo_init, call it.
   MOVQ   _cgo_init(SB), AX
   TESTQ  AX, AX
   JZ needtls
   // arg 1: g0, already in DI
   MOVQ   $setg_gcc<>(SB), SI // arg 2: setg_gcc
#ifdef GOOS_android
   MOVQ   $runtime·tls_g(SB), DX     // arg 3: &tls_g
   // arg 4: TLS base, stored in slot 0 (Android's TLS_SLOT_SELF).
   // Compensate for tls_g (+16).
   MOVQ   -16(TLS), CX
#else
   MOVQ   $0, DX // arg 3, 4: not used when using platform's TLS
   MOVQ   $0, CX
#endif
#ifdef GOOS_windows
   // Adjust for the Win64 calling convention.
   MOVQ   CX, R9 // arg 4
   MOVQ   DX, R8 // arg 3
   MOVQ   SI, DX // arg 2
   MOVQ   DI, CX // arg 1
#endif
   CALL   AX

   // update stackguard after _cgo_init
   MOVQ   $runtime·g0(SB), CX
   MOVQ   (g_stack+stack_lo)(CX), AX
   ADDQ   $const__StackGuard, AX
   MOVQ   AX, g_stackguard0(CX)
   MOVQ   AX, g_stackguard1(CX)

#ifndef GOOS_windows
   JMP ok
#endif
needtls:
#ifdef GOOS_plan9
   // skip TLS setup on Plan 9
   JMP ok
#endif
#ifdef GOOS_solaris
   // skip TLS setup on Solaris
   JMP ok
#endif
#ifdef GOOS_illumos
   // skip TLS setup on illumos
   JMP ok
#endif
#ifdef GOOS_darwin
   // skip TLS setup on Darwin
   JMP ok
#endif

   LEAQ   runtime·m0+m_tls(SB), DI
   CALL   runtime·settls(SB)

   // store through it, to make sure it works
   get_tls(BX)
   MOVQ   $0x123, g(BX)
   MOVQ   runtime·m0+m_tls(SB), AX
   CMPQ   AX, $0x123
   JEQ 2(PC)
   CALL   runtime·abort(SB)
ok:
   // set the per-goroutine and per-mach "registers"
   get_tls(BX)
   LEAQ   runtime·g0(SB), CX
   MOVQ   CX, g(BX)
   LEAQ   runtime·m0(SB), AX

   // save m->g0 = g0
   MOVQ   CX, m_g0(AX)
   // save m0 to g0->m
   MOVQ   AX, g_m(CX)

   CLD             // convention is D is always left cleared
   CALL   runtime·check(SB)

   MOVL   16(SP), AX    // copy argc
   MOVL   AX, 0(SP)
   MOVQ   24(SP), AX    // copy argv
   MOVQ   AX, 8(SP)
   CALL   runtime·args(SB)
   CALL   runtime·osinit(SB)
   CALL   runtime·schedinit(SB)

   // create a new goroutine to start program
   MOVQ   $runtime·mainPC(SB), AX       // entry  mainPC是runtime.main
   PUSHQ  AX		//# newproc的第二个参数入栈，也就是新的goroutine需要执行的函数	
   PUSHQ  $0       // # newproc的第一个参数入栈，该参数表示runtime.main函数需要的参数大小，因为runtime.main没有参数，所以这里是0
   CALL   runtime·newproc(SB)
   POPQ   AX
   POPQ   AX

   // start this M
   CALL   runtime·mstart(SB)

   CALL   runtime·abort(SB)  // mstart should never return
   RET

   // Prevent dead-code elimination of debugCallV1, which is
   // intended to be called by debuggers.
   MOVQ   $runtime·debugCallV1(SB), AX
   RET

DATA   runtime·mainPC+0(SB)/8,$runtime·main(SB)
GLOBL  runtime·mainPC(SB),RODATA,$8
```

```go



func osinit() {
	ncpu = getproccount()
	physHugePageSize = getHugePageSize()
	osArchInit()
}


// g0 m0都是汇编处理

// 启动流程顺序为
// call osinit
// call schedinit
// make & queue new G
// call runtime·mstart
// 新的G调用runtime.main
func schedinit() {
   lockInit(&sched.lock, lockRankSched)
   lockInit(&sched.sysmonlock, lockRankSysmon)
   lockInit(&sched.deferlock, lockRankDefer)
   lockInit(&sched.sudoglock, lockRankSudog)
   lockInit(&deadlock, lockRankDeadlock)
   lockInit(&paniclk, lockRankPanic)
   lockInit(&allglock, lockRankAllg)
   lockInit(&allpLock, lockRankAllp)
   lockInit(&reflectOffs.lock, lockRankReflectOffs)
   lockInit(&finlock, lockRankFin)
   lockInit(&trace.bufLock, lockRankTraceBuf)
   lockInit(&trace.stringsLock, lockRankTraceStrings)
   lockInit(&trace.lock, lockRankTrace)
   lockInit(&cpuprof.lock, lockRankCpuprof)
   lockInit(&trace.stackTab.lock, lockRankTraceStackTab)

   _g_ := getg()
   // 最大线程数量
   sched.maxmcount = 10000

   tracebackinit()
   moduledataverify()
    //内存相关初始化
   stackinit()
   mallocinit()
   fastrandinit() // must run before mcommoninit
    // m相关初始化 把m放到全局allm
   mcommoninit(_g_.m, -1)
    // 查找命令行GODEBUG=
   cpuinit()       // must run before alginit
   alginit()       // maps must not be used before this call
   modulesinit()   // provides activeModules
   typelinksinit() // uses maps, activeModules
   itabsinit()     // uses activeModules

   msigsave(_g_.m)
   initSigmask = _g_.m.sigmask

   goargs()
   goenvs()
   parsedebugvars()
    // gc初始化
   gcinit()
	//netpoll
   sched.lastpoll = uint64(nanotime())
   procs := ncpu
    //初始化p
   if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
      procs = n
   }
    //修改p数量
   if procresize(procs) != nil {
      throw("unknown runnable goroutine during bootstrap")
   }

   // For cgocheck > 1, we turn on the write barrier at all times
   // and check all pointer writes. We can't do this until after
   // procresize because the write barrier needs a P.
   if debug.cgocheck > 1 {
      writeBarrier.cgo = true
      writeBarrier.enabled = true
      for _, p := range allp {
         p.wbBuf.reset()
      }
   }

   if buildVersion == "" {
      // Condition should never trigger. This code just serves
      // to ensure runtime·buildVersion is kept in the resulting binary.
      buildVersion = "unknown"
   }
   if len(modinfo) == 1 {
      // Condition should never trigger. This code just serves
      // to ensure runtime·modinfo is kept in the resulting binary.
      modinfo = ""
   }
}

// Change number of processors. The world is stopped, sched is locked.
// gcworkbufs are not being modified by either the GC or
// the write barrier code.
// Returns list of Ps with local work, they need to be scheduled by the caller.

使用make([]*p, nprocs)初始化全局变量allp，即allp = make([]*p, nprocs)

循环创建并初始化nprocs个p结构体对象并依次保存在allp切片之中

把m0和allp[0]绑定在一起，即m0.p = allp[0], allp[0].m = m0

把除了allp[0]之外的所有p放入到全局变量sched的pidle空闲队列之中
func procresize(nprocs int32) *p {
	old := gomaxprocs
	if old < 0 || nprocs <= 0 {
		throw("procresize: invalid arg")
	}
	if trace.enabled {
		traceGomaxprocs(nprocs)
	}

	// update statistics
	now := nanotime()
	if sched.procresizetime != 0 {
		sched.totaltime += int64(old) * (now - sched.procresizetime)
	}
	sched.procresizetime = now

	// Grow allp if necessary.
	if nprocs > int32(len(allp)) {
		// Synchronize with retake, which could be running
		// concurrently since it doesn't run on a P.
		lock(&allpLock)
		if nprocs <= int32(cap(allp)) {
			allp = allp[:nprocs]
		} else {
            //初始化进这个分支
			nallp := make([]*p, nprocs)
			// Copy everything up to allp's cap so we
			// never lose old allocated Ps.
			copy(nallp, allp[:cap(allp)])
			allp = nallp
		}
		unlock(&allpLock)
	}

	// initialize new P's
	for i := old; i < nprocs; i++ {
		pp := allp[i]
		if pp == nil {
			pp = new(p)
		}
		pp.init(i)
		atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
	}
	
	_g_ := getg()
    // 非p0变更状态
	if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
		// continue to use the current P
		_g_.m.p.ptr().status = _Prunning
		_g_.m.p.ptr().mcache.prepareForSweep()
	} else {
        //初始化时m0->p还未初始化，所以不会执行这个分支
        // 释放当前的p并获取p0
        // 我们必须在做这个在销毁当前p之前，因为p.destory本身有写屏障，因此我们需要一个合法的p
		// release the current P and acquire allp[0].
		if _g_.m.p != 0 {
			//初始化时这里不执行
			_g_.m.p.ptr().m = 0
		}
		_g_.m.p = 0
		p := allp[0]
		p.m = 0
		p.status = _Pidle
        //当前的m绑定p
		acquirep(p)
	}
	
	// g.m.p is now set, so we no longer need mcache0 for bootstrapping.
	mcache0 = nil

	// release resources from unused P's
	for i := nprocs; i < old; i++ {
		p := allp[i]
		p.destroy()
		// can't free P itself because it can be referenced by an M in syscall
	}

	// Trim allp.
	if int32(len(allp)) != nprocs {
		lock(&allpLock)
		allp = allp[:nprocs]
		unlock(&allpLock)
	}

	var runnablePs *p
     //下面这个for 循环把所有空闲的p放入空闲链表
	for i := nprocs - 1; i >= 0; i-- {
		p := allp[i]
       //allp[0]跟m0关联了，所以是不能放入
		if _g_.m.p.ptr() == p {
			continue
		}
		p.status = _Pidle
        // 如果无Local g放入idle
		if runqempty(p) {
            //初始化
			pidleput(p)
		} else {
            //非初始化的时候 绑定g
			p.m.set(mget())
			p.link.set(runnablePs)
			runnablePs = p
		}
	}
	stealOrder.reset(uint32(nprocs))
	var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
	atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
	return runnablePs
}

// mstart是ms的入口
// 必须不能栈分裂因为 我们可能还没设置好栈边界
// 也许运行在STW（因为他还没有一个p）因此写屏障是不允许的
func mstart() {
	_g_ := getg()
	
	osStack := _g_.stack.lo == 0
	if osStack {
        // 初始化栈绑定到系统栈 Cgo可能把栈保存在stack.hi minit也会会更新栈绑定
		size := _g_.stack.hi
		if size == 0 {
			size = 8192 * sys.StackGuardMultiplier
		}
		_g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
		_g_.stack.lo = _g_.stack.hi - size + 1024
	}
    // 初始化栈守卫，然后我们就可以调用go原生的code
	_g_.stackguard0 = _g_.stack.lo + _StackGuard
    // 这是g0，然后我们也可以调用go:systemstack functions，检查stackguard1
	_g_.stackguard1 = _g_.stackguard0
	mstart1()

	//线程退出
	switch GOOS {
	case "windows", "solaris", "illumos", "plan9", "darwin", "aix":
		// Windows, Solaris, illumos, Darwin, AIX and Plan 9 always system-allocate
		// the stack, but put it in _g_.stack before mstart,
		// so the logic above hasn't set osStack yet.
		osStack = true
	}
	mexit(osStack)
}

func mstart1() {
	_g_ := getg()
	//当前应该在系统栈
	if _g_ != _g_.m.g0 {
		throw("bad runtime·mstart")
	}
    // 记录当前调用者的栈顶
    //getcallerpc()获取mstart1执行完的返回地址
    //getcallersp()获取调用mstart1时的栈顶地址
    //save这一行代码非常重要，是我们理解调度循环的关键点之一。这里首先需要注意的是代码中的getcallerpc()返回的是mstart调用mstart1时被call指令压栈的返回地址，getcallersp()函数返回的是调用mstart1函数之前mstart函数的栈顶地址
	save(getcallerpc(), getcallersp())
	asminit()
	minit()

	// Install signal handlers; after minit so that minit can
	// prepare the thread to be able to handle the signals.
    // 如果是m0
	if _g_.m == &m0 {
		mstartm0()
	}
	
	if fn := _g_.m.mstartfn; fn != nil {
        //m0调用runtime.main
		fn()
	}

	if _g_.m != &m0 {
        //不是m0就去绑定p
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
    //进入调度
	schedule()
}
//save函数保存了调度相关的所有信息，包括最为重要的当前正在运行的g的下一条指令的地址和栈顶地址，不管是对g0还是其它goroutine来说这些信息在调度过程中都是必不可少的，
func save(pc, sp uintptr) {
    _g_ := getg()
    _g_.sched.pc = pc //再次运行时的指令地址
    _g_.sched.sp = sp //再次运行时到栈顶
    _g_.sched.lr = 0
    _g_.sched.ret = 0
    _g_.sched.g = guintptr(unsafe.Pointer(_g_))
    // We need to ensure ctxt is zero, but can't have a write
    // barrier here. However, it should always already be zero.
    // Assert that.
    if _g_.sched.ctxt != nil {
        badctxt()
    }
}

// Associate p and the current m.
//
// This function is allowed to have write barriers even if the caller
// isn't because it immediately acquires _p_.
//
//go:yeswritebarrierrec
func acquirep(_p_ *p) {
	// Do the part that isn't allowed to have write barriers.
	wirep(_p_)
	_p_.mcache.prepareForSweep()
}

func wirep(_p_ *p) {
	_g_ := getg()
	//异常
	if _g_.m.p != 0 {
		throw("wirep: already in go")
	}
	if _p_.m != 0 || _p_.status != _Pidle {
		id := int64(0)
		if _p_.m != 0 {
			id = _p_.m.ptr().id
		}
		print("wirep: p->m=", _p_.m, "(", id, ") p->status=", _p_.status, "\n")
		throw("wirep: invalid p state")
	}
	_g_.m.p.set(_p_)
	_p_.m.set(_g_.m)
	_p_.status = _Prunning
}


启动一个sysmon系统监控线程，该线程负责整个程序的gc、抢占调度以及netpoll等功能的监控，在抢占调度一章我们再继续分析sysmon是如何协助完成goroutine的抢占调度的；

执行runtime包的初始化；

执行main包以及main包import的所有包的初始化；

执行main.main函数；

从main.main函数返回后调用exit系统调用退出进程；

// The main goroutine.
func main() {
	g := getg()

	// Racectx of m0->g0 is used only as the parent of the main goroutine.
	// It must not be used for anything else.
	g.m.g0.racectx = 0
    //64位系统上每个goroutine的栈最大可达1G
	if sys.PtrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}

    // 主线程启动 允许newproc创建新的m
	mainStarted = true

	if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
		systemstack(func() {
            //回到g0启动监控线程
			newm(sysmon, nil, -1)
		})
	}

	// Lock the main goroutine onto this, the main OS thread,
	// during initialization. Most programs won't care, but a few
	// do require certain calls to be made by the main thread.
	// Those can arrange for main.main to run in the main thread
	// by calling runtime.LockOSThread during initialization
	// to preserve the lock.
	lockOSThread()

	if g.m != &m0 {
		throw("runtime.main not on m0")
	}

	doInit(&runtime_inittask) // must be before defer
	if nanotime() == 0 {
		throw("nanotime returning zero")
	}

	// Defer unlock so that runtime.Goexit during init does the unlock too.
	needUnlock := true
	defer func() {
		if needUnlock {
			unlockOSThread()
		}
	}()

	// Record when the world started.
	runtimeInitTime = nanotime()
	// 启动gc
	gcenable()

	main_init_done = make(chan bool)
	
	//初始化init
	doInit(&main_inittask)

	close(main_init_done)

	needUnlock = false
	unlockOSThread()

	if isarchive || islibrary {
		// A program compiled with -buildmode=c-archive or c-shared
		// has a main, but it is not executed.
		return
	}
    //用户main
	fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()

	// Make racy client program work: if panicking on
	// another goroutine at the same time as main returns,
	// let the other goroutine finish printing the panic trace.
	// Once it does, it will exit. See issues 3934 and 20018.
	if atomic.Load(&runningPanicDefers) != 0 {
		// Running deferred functions should not take long.
		for c := 0; c < 1000; c++ {
			if atomic.Load(&runningPanicDefers) == 0 {
				break
			}
			Gosched()
		}
	}
	if atomic.Load(&panicking) != 0 {
		gopark(nil, nil, waitReasonPanicWait, traceEvGoStop, 1)
	}

	exit(0)
	for {
		var x *int32
		*x = 0
	}
}
```





![20120907092107290](..\images\20120907092107290.jpg)

mstart1 save后

从上图可以看出，g0.sched.sp指向了mstart1函数执行完成后的返回地址，该地址保存在了mstart函数的栈帧之中；g0.sched.pc指向的是mstart函数中调用mstart1函数之后的 if 语句。