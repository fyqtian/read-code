### bootstrap



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
    // m相关初始化
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
        // 释放当前的p并获取p0
        // 我们必须在做这个在销毁当前p之前，因为p.destory本身有写屏障，因此我们需要一个合法的p
		// release the current P and acquire allp[0].
        //如果不是p0
		if _g_.m.p != 0 {
			if trace.enabled {
				// Pretend that we were descheduled
				// and then scheduled again to keep
				// the trace sane.
				traceGoSched()
				traceProcStop(_g_.m.p.ptr())
			}
 			//挂到m0?
			_g_.m.p.ptr().m = 0
		}
		_g_.m.p = 0
		p := allp[0]
		p.m = 0
		p.status = _Pidle
        //当前的m绑定p
		acquirep(p)
		if trace.enabled {
			traceGoStart()
		}
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
	for i := nprocs - 1; i >= 0; i-- {
		p := allp[i]
		if _g_.m.p.ptr() == p {
			continue
		}
		p.status = _Pidle
        // 如果无Local g放入idle
		if runqempty(p) {
			pidleput(p)
		} else {
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
//go:nosplit
//go:nowritebarrierrec
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
	// Record the caller for use as the top of stack in mcall and
	// for terminating the thread.
	// We're never coming back to mstart1 after we call schedule,
	// so other calls can reuse the current frame.
    // 记录当前调用者的栈顶
    //
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
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
    //进入调度
	schedule()
}


// The main goroutine.
func main() {
	g := getg()

	// Racectx of m0->g0 is used only as the parent of the main goroutine.
	// It must not be used for anything else.
	g.m.g0.racectx = 0

	// Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
	// Using decimal instead of binary GB and MB because
	// they look nicer in the stack overflow failure message.
	if sys.PtrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}

    // 主线程启动 允许newproc创建新的m
	mainStarted = true

	if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
		systemstack(func() {
            //监控线程
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
	if iscgo {
		if _cgo_thread_start == nil {
			throw("_cgo_thread_start missing")
		}
		if GOOS != "windows" {
			if _cgo_setenv == nil {
				throw("_cgo_setenv missing")
			}
			if _cgo_unsetenv == nil {
				throw("_cgo_unsetenv missing")
			}
		}
		if _cgo_notify_runtime_init_done == nil {
			throw("_cgo_notify_runtime_init_done missing")
		}
		// Start the template thread in case we enter Go from
		// a C-created thread and need to create a new thread.
		startTemplateThread()
		cgocall(_cgo_notify_runtime_init_done, nil)
	}

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