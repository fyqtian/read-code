### defer



https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/



我们简单介绍一下 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体中的几个字段：

- `siz` 是参数和结果的内存大小；
- `sp` 和 `pc` 分别代表栈指针和调用方的程序计数器；
- `fn` 是 `defer` 关键字中传入的函数；
- `_panic` 是触发延迟调用的结构体，可能为空；
- `openDefer` 表示当前 `defer` 是否经过开放编码的优化；

```go
// A _defer holds an entry on the list of deferred calls.
// If you add a field here, add code to clear it in freedefer and deferProcStack
// This struct must match the code in cmd/compile/internal/gc/reflect.go:deferstruct
// and cmd/compile/internal/gc/ssa.go:(*state).call.
// Some defers will be allocated on the stack and some on the heap.
// All defers are logically part of the stack, so write barriers to
// initialize them are not required. All defers must be manually scanned,
// and for heap defers, marked.
type _defer struct {
   siz     int32 // includes both arguments and results
   started bool
   heap    bool
   // openDefer indicates that this _defer is for a frame with open-coded
   // defers. We have only one defer record for the entire frame (which may
   // currently have 0, 1, or more defers active).
   openDefer bool
   sp        uintptr  // sp at time of defer
   pc        uintptr  // pc at time of defer
   fn        *funcval // can be nil for open-coded defers
   _panic    *_panic  // panic that is running defer
   link      *_defer

   // If openDefer is true, the fields below record values about the stack
   // frame and associated function that has the open-coded defer(s). sp
   // above will be the sp for the frame, and pc will be address of the
   // deferreturn call in the function.
   fd   unsafe.Pointer // funcdata for the function associated with the frame
   varp uintptr        // value of varp for the stack frame
   // framepc is the current pc associated with the stack frame. Together,
   // with sp above (which is the sp associated with the stack frame),
   // framepc/sp can be used as pc/sp pair to continue a stack trace via
   // gentraceback().
   framepc uintptr
}
```





```go
// Create a new deferred function fn with siz bytes of arguments.
// The compiler turns a defer statement into a call to this.
//go:nosplit
堆上分配
func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
   gp := getg()
   if gp.m.curg != gp {
      // go code on the system stack can't defer
      throw("defer on system stack")
   }

   // the arguments of fn are in a perilous state. The stack map
   // for deferproc does not describe them. So we can't let garbage
   // collection or stack copying trigger until we've copied them out
   // to somewhere safe. The memmove below does that.
   // Until the copy completes, we can only call nosplit routines.
  
   // 调用者栈顶
   sp := getcallersp()
   //defer 调用函数参数地址
   argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
   //调用者 pc寄存器
   callerpc := getcallerpc()
	 //从p找一个 如果没有从全局的找 如果没有申请一个
   d := newdefer(siz)
   if d._panic != nil {
      throw("deferproc: d.panic != nil after newdefer")
   }
   //插到头部
   d.link = gp._defer
   gp._defer = d
   d.fn = fn
   d.pc = callerpc
   d.sp = sp
  // 把func的参数拷贝到_defer后面？
   switch siz {
   case 0:
      // Do nothing.
   case sys.PtrSize:
      *(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
   default:
      memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
   }

   // deferproc returns 0 normally.
   // a deferred func that stops a panic
   // makes the deferproc return 1.
   // the code the compiler generates always
   // checks the return value and jumps to the
   // end of the function if deferproc returns != 0.
   return0()
   // No code can go here - the C return register has
   // been set and must not be clobbered.
}
```



栈上分配

```go
// deferprocStack queues a new deferred function with a defer record on the stack.
// The defer record must have its siz and fn fields initialized.
// All other fields can contain junk.
// The defer record must be immediately followed in memory by
// the arguments of the defer.
// Nosplit because the arguments on the stack won't be scanned
// until the defer record is spliced into the gp._defer list.
//go:nosplit
func deferprocStack(d *_defer) {
   gp := getg()
   if gp.m.curg != gp {
      // go code on the system stack can't defer
      throw("defer on system stack")
   }
   // siz and fn are already set.
   // The other fields are junk on entry to deferprocStack and
   // are initialized here.
   d.started = false
   d.heap = false
   d.openDefer = false
   d.sp = getcallersp()
   d.pc = getcallerpc()
   d.framepc = 0
   d.varp = 0
   // The lines below implement:
   //   d.panic = nil
   //   d.fd = nil
   //   d.link = gp._defer
   //   gp._defer = d
   // But without write barriers. The first three are writes to
   // the stack so they don't need a write barrier, and furthermore
   // are to uninitialized memory, so they must not use a write barrier.
   // The fourth write does not require a write barrier because we
   // explicitly mark all the defer structures, so we don't need to
   // keep track of pointers to them with a write barrier.
   *(*uintptr)(unsafe.Pointer(&d._panic)) = 0
   *(*uintptr)(unsafe.Pointer(&d.fd)) = 0
   *(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
   *(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

   return0()
   // No code can go here - the C return register has
   // been set and must not be clobbered.
}
```



[`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 会从 Goroutine 的 `_defer` 链表中取出最前面的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 并调用 [`runtime.jmpdefer`](https://draveness.me/golang/tree/runtime.jmpdefer) 传入需要执行的函数和参数：

```go
// Run a deferred function if there is one.
// The compiler inserts a call to this at the end of any
// function which calls defer.
// If there is a deferred function, this will call runtime·jmpdefer,
// which will jump to the deferred function such that it appears
// to have been called by the caller of deferreturn at the point
// just before deferreturn was called. The effect is that deferreturn
// is called again and again until there are no more deferred functions.
//
// Declared as nosplit, because the function should not be preempted once we start
// modifying the caller's frame in order to reuse the frame to call the deferred
// function.
//
// The single argument isn't actually used - it just has its address
// taken so it can be matched against pending defers.
//go:nosplit
func deferreturn(arg0 uintptr) {
   gp := getg()
   d := gp._defer
   if d == nil {
      return
   }
   sp := getcallersp()
   //判断是不是当前栈的defer
  // defer里嵌套defer
 // func A(){
   // defer B()
  //}
  //func B(){
   // defer C()
  //}
  
  
   if d.sp != sp {
      return
   }
   // 开放编码
   if d.openDefer {
     //判断是否执行过
      done := runOpenDeferFrame(gp, d)
      if !done {
         throw("unfinished open-coded defers in deferreturn")
      }
      gp._defer = d.link
      freedefer(d)
      return
   }

   // Moving arguments around.
   //
   // Everything called after this point must be recursively
   // nosplit because the garbage collector won't know the form
   // of the arguments until the jmpdefer can flip the PC over to
   // fn.
   
   //复制参数
   switch d.siz {
   case 0:
      // Do nothing.
   case sys.PtrSize:
      *(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
   default:
      memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
   }
   fn := d.fn
   d.fn = nil
   gp._defer = d.link
   //如果是堆上分配的 回收defer
   //本地如果满了 就移动一半到全局 再插入本地
   freedefer(d)
   // If the defer function pointer is nil, force the seg fault to happen
   // here rather than in jmpdefer. gentraceback() throws an error if it is
   // called with a callback on an LR architecture and jmpdefer is on the
   // stack, because the stack trace can be incorrect in that case - see
   // issue #8153).
   _ = fn.fn
   //执行defer
   jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}

runtime.jmpdefer 是一个用汇编语言实现的运行时函数，它的主要工作是跳转到 defer 所在的代码段并在执行结束之后跳转回 runtime.deferreturn。

TEXT runtime·jmpdefer(SB), NOSPLIT, $0-8
	MOVL	fv+0(FP), DX	// fn
	MOVL	argp+4(FP), BX	// caller sp
	LEAL	-4(BX), SP	// caller sp after CALL
#ifdef GOBUILDMODE_shared
	SUBL	$16, (SP)	// return to CALL again
#else
	SUBL	$5, (SP)	// return to CALL again
#endif
	MOVL	0(DX), BX
	JMP	BX	// but first run the deferred function
```





### 开放编码

Go 语言在 1.14 中通过开发编码（Open Coded）实现 `defer` 关键字，该设计使用代码内联优化 `defer` 关键的额外开销并引入函数数据 `funcdata` 管理 `panic` 的调用[3](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#fn:3)，该优化可以将 `defer` 的调用开销从 1.13 版本的 ~35ns 降低至 ~6ns 左右：

而开放编码作为一种优化 `defer` 关键字的方法，它不是在**所有的场景下都会开启的**，开发编码只会在满足以下的条件时启用：

1. 函数的 `defer` 数量少于或者等于 8 个；
2. 函数的 `defer` 关键字**不能在循环中执行**；
3. 函数的 `return` 语句与 `defer` 语句的乘积小于或者等于 15 个；