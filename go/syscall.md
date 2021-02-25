### syscall

https://www.cnblogs.com/flhs/p/12709962.html

https://mp.weixin.qq.com/s?__biz=MzU1OTg5NDkzOA==&mid=2247483840&idx=1&sn=f2d7a78c190ff6ffce829c8938e50fe7&scene=19#wechat_redirect



概念：系统调用为用户态进程提供了硬件的抽象接口。并且是用户空间访问内核的唯一手段，除异常和陷入外，它们是内核唯一的合法入口。保证系统的安全和稳定。

调用号：在Linux中，每个系统调用被赋予一个独一无二的系统调用号。当用户空间的进程执行一个系统调用时，会使用调用号指明系统调用。

syscall指令：因为用户代码特权级较低，无权访问需要最高特权级才能访问的内核地址空间的代码和数据。所以需要特殊指令，在golang中是syscall。



x86-64中通过syscall指令执行系统调用的参数设置

- rax存放系统调用号，调用返回值也会放在rax中
- 当系统调用参数小于等于6个时，参数则须按顺序放到寄存器 rdi，rsi，rdx，r10，r8，r9中。
- 如果系统调用的参数数量大于6个，需将参数保存在一块连续的内存中，并将地址存入rbx中。

```go
func Syscall(trap, a1, a2, a3 uintptr) (r1, r2 uintptr, err Errno) 
func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2 uintptr, err Errno)
func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2 uintptr, err Errno) 
func RawSyscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2 uintptr, err Errno)

// src/syscall/asm_linux_amd64.s
// func Syscall(trap int64, a1, a2, a3 uintptr) (r1, r2, err uintptr);
// Trap # in AX, args in DI SI DX R10 R8 R9, return in AX DX
// Note that this differs from "standard" ABI convention, which
// would pass 4th arg in CX, not R10.

TEXT ·Syscall(SB),NOSPLIT,$0-56
	CALL	runtime·entersyscall(SB)	// 进入系统调用
        // 准备参数，执行系统调用
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	trap+0(FP), AX			// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001		// 对比返回结果
	JLS	ok
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	CALL	runtime·exitsyscall(SB)		// 退出系统调用
	RET
ok:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	CALL	runtime·exitsyscall(SB)		// 退出系统调用
	RET

// func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
TEXT ·Syscall6(SB),NOSPLIT,$0-80
	CALL	runtime·entersyscall(SB)
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	a4+32(FP), R10
	MOVQ	a5+40(FP), R8
	MOVQ	a6+48(FP), R9
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok6
	MOVQ	$-1, r1+56(FP)
	MOVQ	$0, r2+64(FP)
	NEGQ	AX
	MOVQ	AX, err+72(FP)
	CALL	runtime·exitsyscall(SB)
	RET
ok6:
	MOVQ	AX, r1+56(FP)
	MOVQ	DX, r2+64(FP)
	MOVQ	$0, err+72(FP)
	CALL	runtime·exitsyscall(SB)
	RET

// func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2, err uintptr)
TEXT ·RawSyscall(SB),NOSPLIT,$0-56
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok1
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	RET
ok1:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	RET

// func RawSyscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
TEXT ·RawSyscall6(SB),NOSPLIT,$0-80
        ......
	RET
```

**Syscall 与 Syscall6 的区别**：只是参数个数的不同，其他都相同。

**Syscall 与 RawSyscall 的区别**：Syscall 开始会调用 runtime·entersyscall ，结束时会调用 runtime·exitsyscall；而 RawSyscall 没有。这意味着 Syscall 是受调度器控制的，RawSyscall不受。因此 RawSyscall 可能会造成阻塞。



在执行系统调用前调用 entersyscall 和 reentersyscall，reentersyscall的主要功能：

1. 因为要开始系统调用，所以当前G和和P的状态分别变为了 _Gsyscall 和 _Psyscall
2. 而P不会等待M，所以P和M相互解绑
3. 但是M会保留P到 m.oldp 中，在系统调用结束后尝试与P重新绑定。





从上面的分析可以看出，这里**对正在进行系统调用的goroutine的抢占实质上是剥夺与其对应的工作线程所绑定的p**，虽然说处于系统调用之中的工作线程并不需要p，但一旦从操作系统内核返回到用户空间之后就必须绑定一个p才能运行go代码，那么，工作线程从系统调用返回之后如果发现它进入系统调用之前所使用的p被监控线程拿走了，该怎么办呢？接下来我们就来分析这个问题。





```go
func main() {
    fd, err := os.Open("./syscall.go")  //一定会执行系统调用
    if err != nil {
        fmt.Println(err)
    }

    fd.Close()
}


Syscall6函数主要依次干了如下3件事：
调用runtime.entersyscall函数；
使用SYSCALL指令进入系统调用；
调用runtime.exitsyscall函数。
// func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
TEXT ·Syscall6(SB), NOSPLIT, $0-80
    CALL  runtime·entersyscall(SB)

    #按照linux系统约定复制参数到寄存器并调用syscall指令进入内核
    MOVQ  a1+8(FP), DI
    MOVQ  a2+16(FP), SI
    MOVQ  a3+24(FP), DX
    MOVQ  a4+32(FP), R10
    MOVQ  a5+40(FP), R8
    MOVQ  a6+48(FP), R9
    MOVQ  trap+0(FP), AX#syscall entry，系统调用编号放入AX
    SYSCALL  #进入内核

    #从内核返回，判断返回值，linux使用 -1 ~ -4095 作为错误码
    CMPQ  AX, $0xfffffffffffff001
    JLS  ok6

    #系统调用返回错误，为Syscall6函数准备返回值
    MOVQ  $-1, r1+56(FP)
    MOVQ  $0, r2+64(FP)
    NEGQ  AX
    MOVQ  AX, err+72(FP)
    CALL  runtime·exitsyscall(SB)
    RET
ok6:      #系统调用返回错误
    MOVQ  AX, r1+56(FP)
    MOVQ  DX, r2+64(FP)
    MOVQ  $0, err+72(FP)
    CALL  runtime·exitsyscall(SB)
    RET

根据前面的分析和这段代码我们可以猜测，exitsyscall函数将会处理当前工作线程进入系统调用之前所拥有的p被监控线程抢占剥夺的情况

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
禁止线程上发生的抢占，防止出现内存不一致的问题；
保证当前函数不会触发栈分裂或者增长；
保存当前的程序计数器 PC 和栈指针 SP 中的内容；
将 Goroutine 的状态更新至 _Gsyscall；
将 Goroutine 的处理器和线程暂时分离并更新处理器的状态到 _Psyscall；
释放当前线程上的锁；
func reentersyscall(pc, sp uintptr) {
	_g_ := getg() //用户g

	// Disable preemption because during this function g is in Gsyscall status,
	// but can have inconsistent g->sched, do not let GC observe it.
	_g_.m.locks++

	// Entersyscall must not call any function that might split/grow the stack.
	// (See details in comment above.)
	// Catch calls that might, by replacing the stack guard with something that
	// will trip any stack check and leaving a flag to tell newstack to die.
	_g_.stackguard0 = stackPreempt   //抢占标记
	_g_.throwsplit = true

	// Leave SP around for GC and traceback.
    //用来保存g的现场信息，rsp, rbp, rip等等
	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
    //状态变更为系统调用
	casgstatus(_g_, _Grunning, _Gsyscall)
	if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
		systemstack(func() {
			print("entersyscall inconsistent ", hex(_g_.syscallsp), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
			throw("entersyscall")
		})
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
   //p解除与m之间的绑定
	pp.m = 0
    //把p记录在oldp中，等从系统调用返回时，优先绑定这个p
	_g_.m.oldp.set(pp)
    //m解除与p之间的绑定
	_g_.m.p = 0
    //修改当前p的状态，sysmon线程依赖状态实施抢占
	atomic.Store(&pp.status, _Psyscall)
	if sched.gcwaiting != 0 {
		systemstack(entersyscall_gcwait)
		save(pc, sp)
	}

	_g_.m.locks--
}

有sysmon监控线程来抢占剥夺，为什么这里还需要主动解除m和p之间的绑定关系呢？
原因主要在于这里主动解除m和p的绑定关系之后，sysmon线程就不需要通过加锁或cas操作来修改m.p成员从而解除m和p之间的关系；

为什么要记录工作线程进入系统调用之前所绑定的p呢？
因为记录下来可以让工作线程从系统调用返回之后快速找到一个可能可用的p，而不需要加锁从sched的pidle全局队列中去寻找空闲的p。

为什么要把进入系统调用之前所绑定的p搬到m的oldp中，而不是直接使用m的p成员？笔者第一次看到这里也有疑惑，于是翻看了github上的提交记录，从代码作者的提交注释来看，这里主要是从保持m的p成员清晰的语义方面考虑的，因为处于系统调用的m事实上并没有绑定p，所以如果记录在p成员中，p的语义并不够清晰明了。







因为在进入系统调用之前，工作线程调用entersyscall解除了m和p之间的绑定，现在已经从系统调用返回需要重新绑定一个p才能继续运行go代码，所以exitsyscall函数首先就调用exitsyscallfast去尝试绑定一个空闲的p，如果绑定成功则结束exitsyscall函数按函数调用链原路返回去执行其它用户代码，否则则调用mcall函数切换到g0栈执行exitsyscall0函数。下面先来看exitsyscallfast如何尝试绑定一个p，然后再去分析exitsyscall0函数。

exitsyscallfast首先尝试绑定进入系统调用之前所使用的p，如果绑定失败就需要调用exitsyscallfast_pidle去获取空闲的p来绑定。
func exitsyscall() {
	_g_ := getg()
	
	_g_.m.locks++ // see comment in entersyscall
	if getcallersp() > _g_.syscallsp {
		throw("exitsyscall: syscall frame is no longer valid")
	}

	_g_.waitsince = 0
    //系统调用前的P
	oldp := _g_.m.oldp.ptr()
	_g_.m.oldp = 0
    //因为在进入系统调用之前已经解除了m和p之间的绑定，所以现在需要绑定p
    //绑定成功，设置一些状态
	if exitsyscallfast(oldp) {
		 //系统调用完成，增加syscalltick计数，sysmon线程依靠它判断是否是同一次系统调用
		_g_.m.p.ptr().syscalltick++
		 //casgstatus函数会处理一些垃圾回收相关的事情，我们只需知道该函数重新把g设置成_Grunning状态即可
		casgstatus(_g_, _Gsyscall, _Grunning)

		// Garbage collector isn't running (since we are),
		// so okay to clear syscallsp.
		_g_.syscallsp = 0
		_g_.m.locks--
		if _g_.preempt {
			// restore the preemption request in case we've cleared it in newstack
			_g_.stackguard0 = stackPreempt
		} else {
			// otherwise restore the real _StackGuard, we've spoiled it in entersyscall/entersyscallblock
			_g_.stackguard0 = _g_.stack.lo + _StackGuard
		}
		_g_.throwsplit = false
		
		if sched.disable.user && !schedEnabled(_g_) {
			// Scheduling of this goroutine is disabled.
			Gosched()
		}

		return
	}

	_g_.sysexitticks = 0

	_g_.m.locks--

	//没有绑定到p，调用mcall切换到g0栈执行exitsyscall0函数
	mcall(exitsyscall0)

	// Scheduler returned, so we're allowed to run now.
	// Delete the syscallsp information that we left for
	// the garbage collector during the system call.
	// Must wait until now because until gosched returns
	// we don't know for sure that the garbage collector
	// is not running.
	_g_.syscallsp = 0
	_g_.m.p.ptr().syscalltick++
	_g_.throwsplit = false
}


exitsyscallfast首先尝试快速绑定进入系统调用之前所使用的p，因为该p的状态目前还是_Psyscall，监控线程此时可能也正好准备操作这个p的状态，所以这里需要使用cas原子操作来修改状态，保证只有一个线程的cas能够成功，一旦cas操作成功，就表示当前线程获取到了p的使用权，这样当前线程的后续代码就可以直接操作该p了。具体到exitsyscallfast函数，一旦我们拿到p的使用权，就调用wirep把工作线程m和p关联起来，完成绑定工作。所谓的绑定其实就是设置m的p成员指向p和p的m成员指向m。

exitsyscallfast函数如果绑定进入系统调用之前所使用的p失败，则调用exitsyscallfast_pidle从p的全局空闲队列中获取一个p出来绑定，注意这里使用了systemstack(func())函数来调用exitsyscallfast_pidle，systemstack(func())函数有一个func()类型的参数，该函数首先会把栈切换到g0栈，然后调用通过参数传递进来的函数(这里是一个闭包，包含了对exitsyscallfast_pidle函数的调用)，最后再切换回原来的栈并返回，为什么这些代码需要在系统栈也就是g0的栈上执行呢？原则上来说，只要调用链上某个函数有nosplit这个编译器指示就需要在g0栈上去执行，因为有nosplit指示的话编译器就不会插入检查溢出的代码，这样在非g0栈上执行这些nosplit函数就有可能导致栈溢出，g0栈其实就是操作系统线程所使用的栈，它的空间比较大，不需要对runtime代码中的每个函数都做栈溢出检查，否则会严重影响效率。

为什么绑定进入系统调用之前所使用的p会失败呢？原因就在于这个p可能被sysmon监控线程拿走并绑定到其它工作线程，这部分内容我们已经在前面分析过了。
func exitsyscallfast(oldp *p) bool {
	_g_ := getg()

	// Freezetheworld sets stopwait but does not retake P's.
	if sched.stopwait == freezeStopWait {
		return false
	}
    //尝试快速绑定进入系统调用之前所使用的p
	if oldp != nil && oldp.status == _Psyscall && atomic.Cas(&oldp.status, _Psyscall, _Pidle) {
		// There's a cpu for us, so we can run.
		wirep(oldp)
		exitsyscallfast_reacquired()
		return true
	}

	// Try to get any other idle P.
	if sched.pidle != 0 {
		var ok bool
		systemstack(func() {
            //从全局队列中寻找空闲的p，需要加锁，比较慢
			ok = exitsyscallfast_pidle()
			if ok && trace.enabled {
				if oldp != nil {
					// Wait till traceGoSysBlock event is emitted.
					// This ensures consistency of the trace (the goroutine is started after it is blocked).
					for oldp.syscalltick == _g_.m.syscalltick {
						osyield()
					}
				}
				traceGoSysExit(0)
			}
		})
		if ok {
			return true
		}
	}
	return false
}



func exitsyscallfast_pidle() bool {
	lock(&sched.lock)
	_p_ := pidleget()
	if _p_ != nil && atomic.Load(&sched.sysmonwait) != 0 {
        //sysmon 轮询的时候下次唤醒时间>系统时间 并且等待gc或者P全部再空间会进入休眠sched.sysmonwait=1
		atomic.Store(&sched.sysmonwait, 0)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
	if _p_ != nil {
		acquirep(_p_)
		return true
	}
	return false
}


// exitsyscall slow path on g0.
// Failed to acquire P, enqueue gp as runnable.
//
//go:nowritebarrierrec
func exitsyscall0(gp *g) {
	_g_ := getg()
	//用户g状态
	casgstatus(gp, _Gsyscall, _Grunnable)
    //当前工作线程没有绑定到p,所以需要解除m和g的关系
	dropg()
	lock(&sched.lock)
	var _p_ *p
	if schedEnabled(_g_) {
		_p_ = pidleget()
	}
    //没有P放到全局队列
	if _p_ == nil {
		globrunqput(gp)
	} else if atomic.Load(&sched.sysmonwait) != 0 {
		atomic.Store(&sched.sysmonwait, 0)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
    //获取到P 执行
	if _p_ != nil {
		acquirep(_p_)
		execute(gp, false) // Never returns.
	}
    //这段不清楚?
	if _g_.m.lockedg != 0 {
		// Wait until another thread schedules gp and so m again.
		stoplockedm()
		execute(gp, false) // Never returns.
	}
    //当前工作线程进入睡眠，等待被其它线程唤醒
	stopm()
	schedule() // Never returns.
}

对于运行时间过长的goroutine，系统监控线程首先会提出抢占请求，然后工作线程在适当的时候会去响应这个请求并暂停被抢占goroutine的运行，最后工作线程再调用schedule函数继续去调度其它goroutine；

而对于系统调用执行时间过长的goroutine，调度器并没有暂停其执行，只是剥夺了正在执行系统调用的工作线程所绑定的p，要等到工作线程从系统调用返回之后绑定p失败的情况下该goroutine才会真正被暂停运行。
```