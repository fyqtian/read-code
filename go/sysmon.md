### sysmon

https://www.cnblogs.com/flhs/p/12695351.html

https://mp.weixin.qq.com/s?__biz=MzU1OTg5NDkzOA==&mid=2247483835&idx=1&sn=ce6927d36062db1188299295f56ca499&scene=19#wechat_redirect

https://mp.weixin.qq.com/s?__biz=MzU1OTg5NDkzOA==&mid=2247483840&idx=1&sn=f2d7a78c190ff6ffce829c8938e50fe7&scene=19#wechat_redirect

```go
func main() {
        ......
	if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
		systemstack(func() {
			newm(sysmon, nil)
		})
	}
        ......
}

func newm(fn func(), _p_ *p) {
	mp := allocm(_p_, fn)	// 分配一个m
	mp.nextp.set(_p_)
	mp.sigmask = initSigmask
	......
	newm1(mp)
}

func newm1(mp *m) {
	......
	execLock.rlock() // Prevent process clone.
	newosproc(mp)
	execLock.runlock()
}

cloneFlags = _CLONE_VM | /* share memory */
		_CLONE_FS | /* share cwd, etc */
		_CLONE_FILES | /* share fd table */
		_CLONE_SIGHAND | /* share sig handler table */
		_CLONE_SYSVSEM | /* share SysV semaphore undo lists (see issue #20763) */
		_CLONE_THREAD /* revisit - okay for now */

func newosproc(mp *m) {
	stk := unsafe.Pointer(mp.g0.stack.hi)
        ......
	sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
        // 这里注意一下，mstart会被作为工作线程的开始，在runtime.clone中会被调用。
	ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
	sigprocmask(_SIG_SETMASK, &oset, nil)
        ......
}
```

```go
func allocm(_p_ *p, fn func()) *m { {
	_g_ := getg()
	acquirem() // disable GC because it can be called from sysmon
	// 忽略sysmon不会执行的代码
	mp := new(m)	// 新建一个m
	mp.mstartfn = fn	// fn 指向 sysmon
	mcommoninit(mp)	

	if iscgo || GOOS == "solaris" || GOOS == "illumos" || GOOS == "windows" || GOOS == "plan9" || GOOS == "darwin" {
		mp.g0 = malg(-1)
	} else {
		mp.g0 = malg(8192 * sys.StackGuardMultiplier)	// 分配一个g
	}
	mp.g0.m = mp
        ......
	releasem(_g_.m)
	return mp
}
                                   
//调用clone 内核会创建出一个子线程，返回两次。返回0是子线程，否则是父线程
// int32 clone(int32 flags, void *stk, M *mp, G *gp, void (*fn)(void));
                                   
总结一下clone的工作：
准备系统调用clone的参数
将mp，gp，fn从父线程栈复制到寄存器中，给子线程用
调用clone
父线程返回
子线程设置 m.procid、tls、gp，mp互相绑定、调用fn
TEXT runtime·clone(SB),NOSPLIT,$0
        // 准备clone系统调用的参数
	MOVL	flags+0(FP), DI
	MOVQ	stk+8(FP), SI
	MOVQ	$0, DX
	MOVQ	$0, R10

	// 从父进程栈复制mp, gp, fn。子线程会用到。
	MOVQ	mp+16(FP), R8
	MOVQ	gp+24(FP), R9
	MOVQ	fn+32(FP), R12

        // 调用clone
	MOVL	$SYS_clone, AX
	SYSCALL

	// 父线程，返回.
	CMPQ	AX, $0
	JEQ	3(PC)
	MOVL	AX, ret+40(FP)
	RET

	// 子线程，设置栈顶
	MOVQ	SI, SP

	// If g or m are nil, skip Go-related setup.
	CMPQ	R8, $0    // m
	JEQ	nog
	CMPQ	R9, $0    // g
	JEQ	nog

	// 调用系统调用 gettid 获取线程id初始化 mp.procid
	MOVL	$SYS_gettid, AX
	SYSCALL
	MOVQ	AX, m_procid(R8)

	// 设置线程tls
	LEAQ	m_tls(R8), DI
	CALL	runtime·settls(SB)

	// In child, set up new stack
	get_tls(CX)
	MOVQ	R8, g_m(R9)	// gp.m = mp
	MOVQ	R9, g(CX)	// mp.tls[0] = gp
	CALL	runtime·stackcheck(SB)

nog:
	// Call fn
	CALL	R12		// 调用fn，此处是mstart，永不返回。

	// It shouldn't return. If it does, exit that thread.
	MOVL	$111, DI
	MOVL	$SYS_exit, AX
	SYSCALL
	JMP	-3(PC)	// keep exiting
                                   
                                   
func mstart1() {
	_g_ := getg()
	save(getcallerpc(), getcallersp())
	asminit()
	minit()
        // 之前初始化时的调用逻辑是 rt0_go->mstart->mstart1，当时这里的fn == nil。所以会继续向下走，进入调度循环。
        // 现在调用逻辑是通过 newm(sysmon, nil)->allocm 中设置了 mp.mstartfn 为 sysmon的指针。所以下面的 fn 就不是 nil 了
        // fn != nil 调用 sysmon，并且sysmon永不会返回。也就是说不会走到下面schedule中。
	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}
        ......
	schedule()
}
```

```go
监控线程通过在runtime.main中调用newm(sysmon, nil)创建。

newm：调用了allocm 获得了mp。
allocm：new了一个m，也就是前面的mp。并且将 mp.mstartfn 赋值为 sysmon的指针，这很重要，后面会用。
newm->newm1->newosproc->runtime.clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
runtime.clone：准备系统调用clone的参数；从父线程栈复制mp，gp，fn到寄存器；调用clone；父线程返回；子线程设置sp，m.procid，tls，互相绑定mp与gp。调用mstart作为子线程的开始执行。
mstart->mstart1：调用 _g_.m.mstartfn 指向的函数，也就是sysmon，此时监控工作正式开始。



运行计时器 — 获取下一个需要被触发的计时器；
轮询网络 — 获取需要处理的到期文件描述符；
抢占处理器 — 抢占运行时间较长的或者处于系统调用的 Goroutine；
垃圾回收 — 在满足条件时触发垃圾收集回收内存；
// Always runs without a P, so write barriers are not allowed.
//
//go:nowritebarrierrec
func sysmon() {
   lock(&sched.lock)
   sched.nmsys++
    //检查死锁
   checkdead()
   unlock(&sched.lock)

   lasttrace := int64(0)
   idle := 0 // how many cycles in succession we had not wokeup somebody
   delay := uint32(0) // 睡眠时间，开始是20微秒；idle大于50后，翻倍增长；但最大为10毫秒
   for {
      if idle == 0 { // start with 20us sleep...
         delay = 20
      } else if idle > 50 { // start doubling the sleep after 1ms...
         delay *= 2
      }
      if delay > 10*1000 { // up to 10ms
         delay = 10 * 1000
      }
      //睡眠
      usleep(delay)
      //系统时间
      now := nanotime()
      //计时器下一次需要唤醒的时间
      next, _ := timeSleepUntil()
       //当前调度器需要执行垃圾回收或者所有处理器都处于闲置状态时，如果没有需要触发的计时器，那么系统监控可以暂时陷入休眠：
      if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {
         lock(&sched.lock)
         if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
           	//唤醒时间>当前时间
             if next > now {
               atomic.Store(&sched.sysmonwait, 1)
               unlock(&sched.lock)
               // Make wake-up period small enough
               // for the sampling to be correct.
               // 计算强制gc时间
               sleep := forcegcperiod / 2
               if next-now < sleep {
                  sleep = next - now
               }
               shouldRelax := sleep >= osRelaxMinNS
               if shouldRelax {
                  osRelax(true)
               }
               //信号量同步系统监控即将进入休眠的状态
               notetsleep(&sched.sysmonnote, sleep)
               if shouldRelax {
                  osRelax(false)
               }
               //当系统监控被唤醒之后，我们会重新计算当前时间和下一个计时器需要触发的时间、调用 runtime.noteclear 通知系统监控被唤醒并重置休眠的间隔。
                 //如果在这之后，我们发现下一个计时器需要触发的时间小于当前时间，这也说明所有的线程可能正在忙于运行 Goroutine，系统监控会启动新的线程来触发计时器，避免计时器的到期时间有较大的偏差。
               now = nanotime()
               next, _ = timeSleepUntil()
               lock(&sched.lock)
               atomic.Store(&sched.sysmonwait, 0)
               noteclear(&sched.sysmonnote)
            }
            idle = 0
            delay = 20
         }
         unlock(&sched.lock)
      }
      lock(&sched.sysmonlock)
      {
         // If we spent a long time blocked on sysmonlock
         // then we want to update now and next since it's
         // likely stale.
         now1 := nanotime()
         if now1-now > 50*1000 /* 50µs */ {
            next, _ = timeSleepUntil()
         }
         now = now1
      }
		
      //轮询网络 
      // poll network if not polled for more than 10ms
      lastpoll := int64(atomic.Load64(&sched.lastpoll))
      if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
         atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
         //检查待执行的文件描述符并通过 runtime.injectglist 将所有处于就绪状态的 Goroutine 加入全局运行队列中：
         list := netpoll(0) // non-blocking - returns list of goroutines
         if !list.empty() {
            // Need to decrement number of idle locked M's
            // (pretending that one more is running) before injectglist.
            // Otherwise it can lead to the following situation:
            // injectglist grabs all P's but before it starts M's to run the P's,
            // another M returns from syscall, finishes running its G,
            // observes that there is no work to do and no other running M's
            // and reports deadlock.
            incidlelocked(-1)
            injectglist(&list)
            incidlelocked(1)
         }
      }
      if next < now {
         // There are timers that should have already run,
         // perhaps because there is an unpreemptible P.
         // Try to start an M to run them.
         startm(nil, false)
      }
      if atomic.Load(&scavenge.sysmonWake) != 0 {
         // Kick the scavenger awake if someone requested it.
         wakeScavenger()
      }
      // retake P's blocked in syscalls
      // and preempt long running G's
      // 抢占
      if retake(now) != 0 {
         idle = 0
      } else {
         idle++
      }
     
       //在最后，系统监控还会决定是否需要触发强制垃圾回收，runtime.sysmon 会构建 runtime.gcTrigger 并调用 runtime.gcTrigger.test 方法判断是否需要触发垃圾回收：
       //如果需要触发垃圾回收，我们会将用于垃圾回收的 Goroutine 加入全局队列，让调度器选择合适的处理器去执行。
      if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
         lock(&forcegc.lock)
         forcegc.idle = 0
         var list gList
         list.push(forcegc.g)
         injectglist(&list)
         unlock(&forcegc.lock)
      }
      if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
         lasttrace = now
         schedtrace(debug.scheddetail > 0)
      }
      unlock(&sched.sysmonlock)
   }
}

//抢占处理器
//系统监控会在循环中调用 runtime.retake 抢占处于运行或者系统调用中的处理器，该函数会遍历运行时的全局处理器，每个处理器都存储了一个 runtime.sysmontick：

type sysmontick struct {
	schedtick   uint32 //处理器的调度次数
	schedwhen   int64  //处理器上次调度时间
	syscalltick uint32 //系统调用的次数
	syscallwhen int64 //系统调用的时间
}
//当处理器处于 _Prunning 或者 _Psyscall 状态时，如果上一次触发调度的时间已经过去了 10ms，我们会通过 runtime.preemptone 抢占当前处理器；
//当处理器处于 _Psyscall 状态时，在满足以下两种情况下会调用 runtime.handoffp 让出处理器的使用权：
	//当处理器的运行队列不为空或者不存在空闲处理器时
	//当系统调用时间超过了 10ms 
//系统监控通过在循环中抢占处理器来避免同一个 Goroutine 占用线程太长时间造成饥饿问题。
func retake(now int64) uint32 {
	n := 0
	// Prevent allp slice changes. This lock will be completely
	// uncontended unless we're already stopping the world.
	lock(&allpLock)
	// We can't use a range loop over allp because we may
	// temporarily drop the allpLock. Hence, we need to re-fetch
	// allp each time around the loop.
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]
		if _p_ == nil {
			// This can happen if procresize has grown
			// allp but not yet created new Ps.
			continue
		}
		pd := &_p_.sysmontick
		s := _p_.status
		sysretake := false
		if s == _Prunning || s == _Psyscall {
			// 如果运行时间太长，则抢占g
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
				preemptone(_p_)
                // 在系统调用的情况下，preemptone() 不会工作，因为P没有与之关联的M。
				// In case of syscall, preemptone() doesn't
				// work, because there is no M wired to P.
				sysretake = true
			}
		}
// 因为此时P的状态是 _Psyscall，所以是调用过了Syscall(或者Syscall6)开头的 entersyscall 函数，而此函数会解绑P和M，所以 p.m = 0；m.p=0
		if s == _Psyscall {
			// Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
			t := int64(_p_.syscalltick)
			if !sysretake && int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}
			// On the one hand we don't want to retake Ps if there is no other work to do,
			// but on the other hand we want to retake them eventually
			// because they can prevent the sysmon thread from deep sleep.
             // p的local队列为空 && （存在自旋的m || 存在空闲的p） && 距离上次系统调用不超过10ms ==> 不需要继续执行
			// 只要满足下面三个条件中的任意一个，则抢占该p，否则不抢占
            // 1. p的运行队列里面有等待运行的goroutine
            // 2. 没有无所事事的p
            // 3. 从上一次监控线程观察到p对应的m处于系统调用之中到现在已经超过10了毫秒
            if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			// Drop allpLock so we can take sched.lock.
			unlock(&allpLock)
			// Need to decrement number of idle locked M's
			// (pretending that one more is running) before the CAS.
			// Otherwise the M from which we retake can exit the syscall,
			// increment nmidle and report deadlock.
			incidlelocked(-1)
            //状态改为空闲
			if atomic.Cas(&_p_.status, s, _Pidle) {
				n++
				_p_.syscalltick++
                // 尝试为p寻找一个m（startm），如果没有寻找到则 pidleput
				handoffp(_p_)
			}
			incidlelocked(1)
			lock(&allpLock)
		}
	}
	unlock(&allpLock)
	return uint32(n)
}


// Tell the goroutine running on processor P to stop.
// This function is purely best-effort. It can incorrectly fail to inform the
// goroutine. It can send inform the wrong goroutine. Even if it informs the
// correct goroutine, that goroutine might ignore the request if it is
// simultaneously executing newstack.
// No lock needs to be held.
// Returns true if preemption request was issued.
// The actual preemption will happen at some point in the future
// and will be indicated by the gp->status no longer being
// Grunning
func preemptone(_p_ *p) bool {
	mp := _p_.m.ptr()
	if mp == nil || mp == getg().m {
		return false
	}
	gp := mp.curg
    // 如果是nil或是m.g0
	if gp == nil || gp == mp.g0 {
		return false
	}
	// 设置抢占标记
	gp.preempt = true

	// Every call in a go routine checks for stack overflow by
	// comparing the current stack pointer to gp->stackguard0.
	// Setting gp->stackguard0 to StackPreempt folds
	// preemption into the normal stack overflow check.
    // 设置为一个大于任何真实sp的值。
	gp.stackguard0 = stackPreempt

	// Request an async preemption of this P.
    // 基于信号的异步的抢占调度
	if preemptMSupported && debug.asyncpreemptoff == 0 {
		_p_.preempt = true
		preemptM(mp)
	}

	return true
}

协作式抢占调度 golang的编译器一般会在函数的汇编代码前后自动添加栈是否需要扩张的检查代码。


   0x0000000000458360 <+0>:     mov    %fs:0xfffffffffffffff8,%rcx  # 将当前g的指针存入rcx。tls还记得么？
   0x0000000000458369 <+9>:     cmp    0x10(%rcx),%rsp              # 比较g.stackguard0和rsp。g结构体地址偏移16个字节就是g.stackguard0。
   0x000000000045836d <+13>:    jbe    0x4583b0 <main.caller+80>    # 如果rsp较小,表示栈有溢出风险,调用runtime.morestack_noctxt
   // 此处省略具体函数汇编代码
   0x00000000004583b0 <+80>:	callq  0x451b30 <runtime.morestack_noctxt>
   0x00000000004583b5 <+85>:	jmp    0x458360 <main.caller>


假设上面的汇编代码是属于一个叫 caller 的函数的（实际上确实是的）。

当运行caller的G（暂且称其为gp）由于运行时间过长，被监控线程sysmon通过preemptone函数标记其 gp.preempt = true；gp.stackguard0 = stackPreempt。

当caller被调用时，会先进行栈的检查，因为 stackPreempt 是一个大于任何真实sp的值，所以jbe指令跳转调用 runtime.morestack_noctxt 。

goschedImpl是抢占调度的关键逻辑，从 morestack_noctxt 到 goschedImpl 的调用链如下：

morestack_noctxt->morestack->newstack->gopreempt_m->goschedImpl。其中 morestack_noctxt 和 morestack 由汇编编写。

TEXT runtime·morestack(SB),NOSPLIT,$0-0
	// Cannot grow scheduler stack (m->g0).
	get_tls(CX)
	MOVQ	g(CX), BX
	MOVQ	g_m(BX), BX
	MOVQ	m_g0(BX), SI
	CMPQ	g(CX), SI
	JNE	3(PC)
	CALL	runtime·badmorestackg0(SB)
	CALL	runtime·abort(SB)

	// Cannot grow signal stack (m->gsignal).
	MOVQ	m_gsignal(BX), SI
	CMPQ	g(CX), SI
	JNE	3(PC)
	CALL	runtime·badmorestackgsignal(SB)
	CALL	runtime·abort(SB)

	// Called from f.
	// Set m->morebuf to f's caller.
	NOP	SP	// tell vet SP changed - stop checking offsets
	MOVQ	8(SP), AX	// f's caller's PC
	MOVQ	AX, (m_morebuf+gobuf_pc)(BX)
	LEAQ	16(SP), AX	// f's caller's SP
	MOVQ	AX, (m_morebuf+gobuf_sp)(BX)
	get_tls(CX)
	MOVQ	g(CX), SI
	MOVQ	SI, (m_morebuf+gobuf_g)(BX)

	// Set g->sched to context in f.
	MOVQ	0(SP), AX // f's PC
	MOVQ	AX, (g_sched+gobuf_pc)(SI)
	MOVQ	SI, (g_sched+gobuf_g)(SI)
	LEAQ	8(SP), AX // f's SP
	MOVQ	AX, (g_sched+gobuf_sp)(SI)
	MOVQ	BP, (g_sched+gobuf_bp)(SI)
	MOVQ	DX, (g_sched+gobuf_ctxt)(SI)

	// Call newstack on m->g0's stack.
	MOVQ	m_g0(BX), BX
	MOVQ	BX, g(CX)
	MOVQ	(g_sched+gobuf_sp)(BX), SP
	CALL	runtime·newstack(SB)
	CALL	runtime·abort(SB)	// crash if newstack returns
	RET


// Called from runtime·morestack when more stack is needed.
// Allocate larger stack and relocate to new stack.
// Stack growth is multiplicative, for constant amortized cost.
//
// g->atomicstatus will be Grunning or Gscanrunning upon entry.
// If the scheduler is trying to stop this g, then it will set preemptStop.
//
// This must be nowritebarrierrec because it can be called as part of
// stack growth from other nowritebarrierrec functions, but the
// compiler doesn't check this.
//
//go:nowritebarrierrec
func newstack() {
	thisg := getg()
	// TODO: double check all gp. shouldn't be getg().
	if thisg.m.morebuf.g.ptr().stackguard0 == stackFork {
		throw("stack growth after fork")
	}
	if thisg.m.morebuf.g.ptr() != thisg.m.curg {
		print("runtime: newstack called from g=", hex(thisg.m.morebuf.g), "\n"+"\tm=", thisg.m, " m->curg=", thisg.m.curg, " m->g0=", thisg.m.g0, " m->gsignal=", thisg.m.gsignal, "\n")
		morebuf := thisg.m.morebuf
		traceback(morebuf.pc, morebuf.sp, morebuf.lr, morebuf.g.ptr())
		throw("runtime: wrong goroutine in newstack")
	}

	gp := thisg.m.curg

	if thisg.m.curg.throwsplit {
		// Update syscallsp, syscallpc in case traceback uses them.
		morebuf := thisg.m.morebuf
		gp.syscallsp = morebuf.sp
		gp.syscallpc = morebuf.pc
		pcname, pcoff := "(unknown)", uintptr(0)
		f := findfunc(gp.sched.pc)
		if f.valid() {
			pcname = funcname(f)
			pcoff = gp.sched.pc - f.entry
		}
		print("runtime: newstack at ", pcname, "+", hex(pcoff),
			" sp=", hex(gp.sched.sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
			"\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
			"\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")

		thisg.m.traceback = 2 // Include runtime frames
		traceback(morebuf.pc, morebuf.sp, morebuf.lr, gp)
		throw("runtime: stack split at bad time")
	}

	morebuf := thisg.m.morebuf
	thisg.m.morebuf.pc = 0
	thisg.m.morebuf.lr = 0
	thisg.m.morebuf.sp = 0
	thisg.m.morebuf.g = 0

	// NOTE: stackguard0 may change underfoot, if another thread
	// is about to try to preempt gp. Read it just once and use that same
	// value now and below.
	preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

	// Be conservative about where we preempt.
	// We are interested in preempting user Go code, not runtime code.
	// If we're holding locks, mallocing, or preemption is disabled, don't
	// preempt.
	// This check is very early in newstack so that even the status change
	// from Grunning to Gwaiting and back doesn't happen in this case.
	// That status change by itself can be viewed as a small preemption,
	// because the GC might change Gwaiting to Gscanwaiting, and then
	// this goroutine has to wait for the GC to finish before continuing.
	// If the GC is in some way dependent on this goroutine (for example,
	// it needs a lock held by the goroutine), that small preemption turns
	// into a real deadlock.
	if preempt {
		if !canPreemptM(thisg.m) {
			// Let the goroutine keep running for now.
			// gp->preempt is set, so it will be preempted next time.
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	if gp.stack.lo == 0 {
		throw("missing stack in newstack")
	}
	sp := gp.sched.sp
	if sys.ArchFamily == sys.AMD64 || sys.ArchFamily == sys.I386 || sys.ArchFamily == sys.WASM {
		// The call to morestack cost a word.
		sp -= sys.PtrSize
	}
	if stackDebug >= 1 || sp < gp.stack.lo {
		print("runtime: newstack sp=", hex(sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
			"\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
			"\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")
	}
	if sp < gp.stack.lo {
		print("runtime: gp=", gp, ", goid=", gp.goid, ", gp->status=", hex(readgstatus(gp)), "\n ")
		print("runtime: split stack overflow: ", hex(sp), " < ", hex(gp.stack.lo), "\n")
		throw("runtime: split stack overflow")
	}

	if preempt {
		if gp == thisg.m.g0 {
			throw("runtime: preempt g0")
		}
		if thisg.m.p == 0 && thisg.m.locks == 0 {
			throw("runtime: g is running but p is not")
		}

		if gp.preemptShrink {
			// We're at a synchronous safe point now, so
			// do the pending stack shrink.
			gp.preemptShrink = false
			shrinkstack(gp)
		}

		if gp.preemptStop {
			preemptPark(gp) // never returns
		}

		// Act like goroutine called runtime.Gosched.
		gopreempt_m(gp) // never return
	}

	// Allocate a bigger segment and move the stack.
	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize * 2

	// Make sure we grow at least as much as needed to fit the new frame.
	// (This is just an optimization - the caller of morestack will
	// recheck the bounds on return.)
	if f := findfunc(gp.sched.pc); f.valid() {
		max := uintptr(funcMaxSPDelta(f))
		for newsize-oldsize < max+_StackGuard {
			newsize *= 2
		}
	}

	if newsize > maxstacksize {
		print("runtime: goroutine stack exceeds ", maxstacksize, "-byte limit\n")
		print("runtime: sp=", hex(sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n")
		throw("stack overflow")
	}

	// The goroutine must be executing in order to call newstack,
	// so it must be Grunning (or Gscanrunning).
	casgstatus(gp, _Grunning, _Gcopystack)

	// The concurrent GC will not scan the stack while we are doing the copy since
	// the gp is in a Gcopystack status.
	copystack(gp, newsize)
	if stackDebug >= 1 {
		print("stack grow done\n")
	}
	casgstatus(gp, _Gcopystack, _Grunning)
	gogo(&gp.sched)
}

func gopreempt_m(gp *g) {
	goschedImpl(gp)
}
func goschedImpl(gp *g) {
	status := readgstatus(gp)
	if status&^_Gscan != _Grunning {
		dumpgstatus(gp)
		throw("bad g status")
	}
	casgstatus(gp, _Grunning, _Grunnable)
	dropg()
	lock(&sched.lock)
	globrunqput(gp)
	unlock(&sched.lock)

	schedule()
}


基于信号的异步抢占

func preemptM(mp *m) {
	......
	signalM(mp, sigPreempt)
}
// signalM sends a signal to mp.
func signalM(mp *m, sig int) {
	if atomic.Load(&touchStackBeforeSignal) != 0 {
		atomic.Cas((*uint32)(unsafe.Pointer(mp.gsignal.stack.hi-4)), 0, 0)
	}
	tgkill(getpid(), int(mp.procid), sig)
}
TEXT ·tgkill(SB),NOSPLIT,$0
	MOVQ	tgid+0(FP), DI
	MOVQ	tid+8(FP), SI
	MOVQ	sig+16(FP), DX
	MOVL	$SYS_tgkill, AX
	SYSCALL
	RET




内核在收到 _SIGURG 信号后，会调用该线程注册的信号处理程序，最终会执行到以下程序。
因为注册逻辑不是问题的关注核心，所以就放在后面有介绍。

func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
	_g_ := getg()
	c := &sigctxt{info, ctxt}
        ......
	if sig == sigPreempt {    // const sigPreempt
		doSigPreempt(gp, c)
	}
        ......
}

func doSigPreempt(gp *g, ctxt *sigctxt) {
	// Check if this G wants to be preempted and is safe to
	// preempt.
	if wantAsyncPreempt(gp) && isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()) {
		// Inject a call to asyncPreempt.
		ctxt.pushCall(funcPC(asyncPreempt))
	}

	// Acknowledge the preemption.
	atomic.Xadd(&gp.m.preemptGen, 1)
}

func (c *sigctxt) pushCall(targetPC uintptr) {
	// Make it look like the signaled instruction called target.
	pc := uintptr(c.rip())
	sp := uintptr(c.rsp())
	sp -= sys.PtrSize
	*(*uintptr)(unsafe.Pointer(sp)) = pc
	c.set_rsp(uint64(sp))
	c.set_rip(uint64(targetPC)) // pc指向asyncPreempt
}

// asyncPreempt->asyncPreempt2
func asyncPreempt2() {
	gp := getg()
	gp.asyncSafePoint = true
	if gp.preemptStop {
		mcall(preemptPark)
	} else {
		mcall(gopreempt_m)
	}
	gp.asyncSafePoint = false
}

// gopreempt_m里调用了goschedImpl，这个函数上面分析过，是完成抢占的关键。此时也就是完成了抢占，进入调度循环。
func gopreempt_m(gp *g) {
	if trace.enabled {
		traceGoPreempt()
	}
	goschedImpl(gp)
}



信号处理程序的注册与执行#
注册#
m0的信号处理程序是在整个程序一开始就在 mstart1 中开始注册的。

而其他M所属线程因为在clone的时候指定了 _CLONE_SIGHAND 标记，共享了信号handler table。所以一出生就有了。

注册逻辑如下：

// 省略了一些无关代码
func mstart1() {
	if _g_.m == &m0 {
		mstartm0()
	}
}

func mstartm0() {
	initsig(false)
}

// 循环注册信号处理程序
func initsig(preinit bool) {
	for i := uint32(0); i < _NSIG; i++ {
                ......
		setsig(i, funcPC(sighandler))
	}
}

// sigtramp注册为处理程序
func setsig(i uint32, fn uintptr) {
	var sa sigactiont
	sa.sa_flags = _SA_SIGINFO | _SA_ONSTACK | _SA_RESTORER | _SA_RESTART
	sigfillset(&sa.sa_mask)
	if GOARCH == "386" || GOARCH == "amd64" {
		sa.sa_restorer = funcPC(sigreturn)
	}
	if fn == funcPC(sighandler) {
		if iscgo {
			fn = funcPC(cgoSigtramp)
		} else {
			fn = funcPC(sigtramp)
		}
	}
	sa.sa_handler = fn
	sigaction(i, &sa, nil)
}

// sigaction->sysSigaction->rt_sigaction
// 调用rt_sigaction系统调用，注册处理程序
TEXT runtime·rt_sigaction(SB),NOSPLIT,$0-36
	MOVQ	sig+0(FP), DI
	MOVQ	new+8(FP), SI
	MOVQ	old+16(FP), DX
	MOVQ	size+24(FP), R10
	MOVL	$SYS_rt_sigaction, AX
	SYSCALL
	MOVL	AX, ret+32(FP)
	RET

以上逻辑主要作用就是循环注册 _NSIG（65） 个信号处理程序，其实都是 sigtramp 函数。操作系统内核在收到信号后会调用此函数。


TEXT runtime·sigtramp(SB),NOSPLIT,$72
        ......
	MOVQ	DX, ctx-56(SP)
	MOVQ	SI, info-64(SP)
	MOVQ	DI, signum-72(SP)
	MOVQ	$runtime·sigtrampgo(SB), AX
	CALL AX
        ......
	RET

func sigtrampgo(sig uint32, info *siginfo, ctx unsafe.Pointer) {
        ......
        c := &sigctxt{info, ctx}
	g := sigFetchG(c) // getg()
        ......
	sighandler(sig, info, ctx, g)
        ......
}

```



### 检查死锁

```go
// Check for deadlock situation.
// The check is based on number of running M's, if 0 -> deadlock.
// sched.lock must be held.
func checkdead() {

   // for it. (It is possible to have an extra M on Windows without cgo to
   // accommodate callbacks created by syscall.NewCallback. See issue #6751
   // for details.)
   var run0 int32
   if !iscgo && cgoHasExtraM {
      mp := lockextra(true)
      haveExtraM := extraMCount > 0
      unlockextra(mp)
      if haveExtraM {
         run0 = 1
      }
   }
    //runtime.mcount() 根据下一个待创建的线程 id 和释放的线程数得到系统中存在的线程数；
    //nmidle 是处于空闲状态的线程数量
    //nmidlelocked 是处于锁定状态的线程数量；
    //nmsys 是处于系统调用的线程数量；
    //利用上述几个线程相关数据，我们可以得到正在运行的线程数，如果线程数量大于 0，说明当前程序不存在死锁；如果线程数小于 0，说明当前程序的状态不一致；如果线程数等于 0，我们需要进一步检查程序的运行状态：
   run := mcount() - sched.nmidle - sched.nmidlelocked - sched.nmsys
   if run > run0 {
      return
   }
   if run < 0 {
      print("runtime: checkdead: nmidle=", sched.nmidle, " nmidlelocked=", sched.nmidlelocked, " mcount=", mcount(), " nmsys=", sched.nmsys, "\n")
      throw("checkdead: inconsistent counts")
   }

   grunning := 0
   lock(&allglock)
    //当存在 Goroutine 处于 _Grunnable、_Grunning 和 _Gsyscall 状态时，意味着程序发生了死锁；
    //当所有的 Goroutine 都处于 _Gidle、_Gdead 和 _Gcopystack 状态时，意味着主程序调用了 runtime.goexit；
   for i := 0; i < len(allgs); i++ {
      gp := allgs[i]
      if isSystemGoroutine(gp, false) {
         continue
      }
      s := readgstatus(gp)
      switch s &^ _Gscan {
      case _Gwaiting,
         _Gpreempted:
         grunning++
      case _Grunnable,
         _Grunning,
         _Gsyscall:
         unlock(&allglock)
         print("runtime: checkdead: find g ", gp.goid, " in status ", s, "\n")
         throw("checkdead: runnable g")
      }
   }
   unlock(&allglock)
   if grunning == 0 { // possible if main goroutine calls runtime·Goexit()
      unlock(&sched.lock) // unlock so that GODEBUG=scheddetail=1 doesn't hang
      throw("no goroutines (main called runtime.Goexit) - deadlock!")
   }
	//当运行时存在等待的 Goroutine 并且不存在正在运行的 Goroutine 时，我们会检查处理器中存在的计时器

   // Maybe jump time forward for playground.
   if faketime != 0 {
      when, _p_ := timeSleepUntil()
      if _p_ != nil {
         faketime = when
         for pp := &sched.pidle; *pp != 0; pp = &(*pp).ptr().link {
            if (*pp).ptr() == _p_ {
               *pp = _p_.link
               break
            }
         }
         mp := mget()
         if mp == nil {
            // There should always be a free M since
            // nothing is running.
            throw("checkdead: no m for timer")
         }
         mp.nextp.set(_p_)
         notewakeup(&mp.park)
         return
      }
   }

   // There are no goroutines running, so we can look at the P's.
	//如果处理器中存在等待的计时器，那么所有的 Goroutine 陷入休眠状态是合理的，不过如果不存在等待的计时器，运行时会直接报错并退出程序。
   for _, _p_ := range allp {
      if len(_p_.timers) > 0 {
         return
      }
   }

   getg().m.throwing = -1 // do not dump full stacks
   unlock(&sched.lock)    // unlock so that GODEBUG=scheddetail=1 doesn't hang
   throw("all goroutines are asleep - deadlock!")
}
```





### 网络轮询

该函数会将所有 Goroutine 的状态从 `_Gwaiting` 切换至 `_Grunnable` 并加入全局运行队列等待运行，如果当前程序中存在空闲的处理器，会通过 [`runtime.startm`](https://draveness.me/golang/tree/runtime.startm) 启动线程来执行这些任务。

```go
func injectglist(glist *gList) {
	if glist.empty() {
		return
	}
	lock(&sched.lock)
	var n int
	for n = 0; !glist.empty(); n++ {
		gp := glist.pop()
		casgstatus(gp, _Gwaiting, _Grunnable)
		globrunqput(gp)
	}
	unlock(&sched.lock)
	for ; n != 0 && sched.npidle != 0; n-- {
		startm(nil, false)
	}
	*glist = gList{}
}
```



```go
handoffp函数流程比较简单，它的主要任务是通过各种条件判断是否需要启动工作线程来接管_p_，如果不需要则把_p_放入P的全局空闲队列。

从handoffp的代码可以看出，在如下几种情况下则需要调用我们已经分析过的startm函数启动新的工作线程出来接管_p_：

_p_的本地运行队列或全局运行队列里面有待运行的goroutine；

需要帮助gc完成标记工作；

系统比较忙，所有其它_p_都在运行goroutine，需要帮忙；

所有其它P都已经处于空闲状态，如果需要监控网络连接读写事件，则需要启动新的m来poll网络连接。

到此，sysmon监控线程对处于系统调用之中的p的抢占就已经完成。

// Hands off P from syscall or locked M.
// Always runs without a P, so write barriers are not allowed.
//go:nowritebarrierrec
func handoffp(_p_ *p) {
   // handoffp must start an M in any situation where
   // findrunnable would return a G to run on _p_.

    //运行队列不为空，需要启动m来接管
   if !runqempty(_p_) || sched.runqsize != 0 {
      startm(_p_, false)
      return
   }
    //有垃圾回收工作需要做，也需要启动m来接管
   if gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) {
      startm(_p_, false)
      return
   }
   // no local work, check that there are no spinning/idle M's,
   // otherwise our help is not required
     //所有其它p都在运行goroutine，说明系统比较忙，需要启动m
   if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 && atomic.Cas(&sched.nmspinning, 0, 1) { // TODO: fast atomic
      startm(_p_, true)
      return
   }
   lock(&sched.lock)
    //如果gc正在等待Stop The World
   if sched.gcwaiting != 0 {
      _p_.status = _Pgcstop
      sched.stopwait--
      if sched.stopwait == 0 {
          //唤醒
         notewakeup(&sched.stopnote)
      }
      unlock(&sched.lock)
      return
   }
   if _p_.runSafePointFn != 0 && atomic.Cas(&_p_.runSafePointFn, 1, 0) {
      sched.safePointFn(_p_)
      sched.safePointWait--
      if sched.safePointWait == 0 {
         notewakeup(&sched.safePointNote)
      }
   }
  //全局运行队列有工作要做
   if sched.runqsize != 0 {
      unlock(&sched.lock)
      startm(_p_, false)
      return
   }
   // If this is the last running P and nobody is polling network,
   // need to wakeup another M to poll network.
     //不能让所有的p都空闲下来，因为需要监控网络连接读写事件
   if sched.npidle == uint32(gomaxprocs-1) && atomic.Load64(&sched.lastpoll) != 0 {
      unlock(&sched.lock)
      startm(_p_, false)
      return
   }
   if when := nobarrierWakeTime(_p_); when != 0 {
      wakeNetPoller(when)
   }
     //无事可做，把p放入全局空闲队列
   pidleput(_p_)
   unlock(&sched.lock)
}
```