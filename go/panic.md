### panic

https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/

- `panic` 能够改变程序的控制流，调用 `panic` 后会立刻停止执行当前函数的剩余代码，并在当前 Goroutine 中递归执行调用方的 `defer`；
- `recover` 可以中止 `panic` 造成的程序崩溃。**它是一个只能在 `defer` 中发挥作用的函数**，在其他作用域中调用不会发挥作用



## 现象

我们先通过几个例子了解一下使用 `panic` 和 `recover` 关键字时遇到的现象，部分现象也与上一节分析的 `defer` 关键字有关：

- `panic` 只会触发当前 Goroutine 的 `defer`；
- `recover` **只有在 `defer` 中调用才会生效**；
- `panic` **允许在 `defer` 中嵌套多次调用**；

### 跨协程失效

首先要介绍的现象是 `panic` **只会触发当前 Goroutine 的延迟函数调用**，我们可以通过如下所示的代码了解该现象：

```go
func main() {
	defer println("in main")
	go func() {
		defer println("in goroutine")
		panic("")
	}()

	time.Sleep(1 * time.Second)
}

$ go run main.go
in goroutine
panic:
...
```



### 失效的崩溃恢复

初学 Go 语言的读者可能会写出下面的代码，在主程序中调用 `recover` 试图中止程序的崩溃，但是从运行的结果中我们也能看出，下面的程序没有正常退出。

仔细分析一下这个过程就能理解这种现象背后的原因，**`recover` 只有在发生 `panic` 之后调用才会生效**。然而在上面的控制流中，`recover` 是在 `panic` 之前调用的，并不满足生效的条件，所以我们需要在 `defer` 中使用 `recover` 关键字。

```go
func main() {
	defer fmt.Println("in main")
	if err := recover(); err != nil {
		fmt.Println(err)
	}

	panic("unknown err")
}

$ go run main.go
in main
panic: unknown err

goroutine 1 [running]:
main.main()
	...
exit status 2


func main(){
  //recover也不能生效 还没搞明白
  // p.argp is the argument pointer of that topmost deferred function call.
  // if p != nil && !p.goexit && !p.recovered && argp == uintptr(p.argp) 
  defer recover()
  panic("")
  
}
```







### 嵌套崩溃

Go 语言中的 `panic` 是可以多次嵌套调用的。一些熟悉 Go 语言的读者很可能也不知道这个知识点，如下所示的代码就展示了如何在 `defer` 函数中多次调用 `panic`：

从上述程序输出的结果，我们可以确定程序多次调用 `panic` 也不会影响 `defer` 函数的正常执行，所以使用 `defer` 进行收尾工作一般来说都是安全的。

```go
func main() {
	defer fmt.Println("in main")
	defer func() {
		defer func() {
			panic("panic again and again")
		}()
		panic("panic again")
	}()

	panic("panic once")
}

$ go run main.go
in main
panic: panic once
	panic: panic again
	panic: panic again and again

goroutine 1 [running]:
...
exit status 2
```





```go
// A _panic holds information about an active panic.
//
// This is marked go:notinheap because _panic values must only ever
// live on the stack.
//
// The argp and link fields are stack pointers, but don't need special
// handling during stack growth: because they are pointer-typed and
// _panic values only live on the stack, regular stack pointer
// adjustment takes care of them.
//
//go:notinheap

argp 是指向 defer 调用时参数的指针；
arg 是调用 panic 时传入的参数；
link 指向了更早调用的 runtime._panic 结构；
recovered 表示当前 runtime._panic 是否被 recover 恢复；
aborted 表示当前的 panic 是否被强行终止；

结构体中的 pc、sp 和 goexit 三个字段都是为了修复 runtime.Goexit 带来的问题引入的。runtime.Goexit 能够只结束调用该函数的 Goroutine 而不影响其他的 Goroutine，但是该函数会被 defer 中的 panic 和 recover 取消，引入这三个字段就是为了保证该函数的一定会生效。

type _panic struct {
   argp      unsafe.Pointer // pointer to arguments of deferred call run during panic; cannot move - known to liblink
   arg       interface{}    // argument to panic
   link      *_panic        // link to earlier panic
   pc        uintptr        // where to return to in runtime if this panic is bypassed
   sp        unsafe.Pointer // where to return to in runtime if this panic is bypassed
   recovered bool           // whether this panic is over
   aborted   bool           // the panic was aborted
   goexit    bool
}


// The implementation of the predeclared function panic.

创建新的 runtime._panic 并添加到所在 Goroutine 的 _panic 链表的最前面；
在循环中不断从当前 Goroutine 的 _defer 中链表获取 runtime._defer 并调用 runtime.reflectcall 运行延迟调用函数；
调用 runtime.fatalpanic 中止整个程序；
func gopanic(e interface{}) {
	gp := getg()
	
	var p _panic
	p.arg = e
	p.link = gp._panic
  //当前panic插入到头部
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	atomic.Xadd(&runningPanicDefers, 1)

	// By calculating getcallerpc/getcallersp here, we avoid scanning the
	// gopanic frame (stack scanning is slow...)
	addOneOpenDeferFrame(gp, getcallerpc(), unsafe.Pointer(getcallersp()))

	for {
		d := gp._defer
    // 如果没有defer跳出
		if d == nil {
			break
		}

		// If defer was started by earlier panic or Goexit (and, since we're back here, that triggered a new panic),
		// take defer off list. An earlier panic will not continue running, but we will make sure below that an
		// earlier Goexit does continue running.
    
    // 如果defer被先前的panic或goexit触发
    
		if d.started {
			if d._panic != nil {
        //状态标记为终止
				d._panic.aborted = true
			}
			d._panic = nil
			if !d.openDefer {
				// For open-coded defers, we need to process the
				// defer again, in case there are any other defers
				// to call in the frame (not including the defer
				// call that caused the panic).
				d.fn = nil
				gp._defer = d.link
				freedefer(d)
				continue
			}
		}

		// Mark defer as started, but keep on list, so that traceback
		// can find and update the defer's argument frame if stack growth
		// or a garbage collection happens before reflectcall starts executing d.fn.
    
    //标记panic开始
		d.started = true

		// Record the panic that is running the defer.
		// If there is a new panic during the deferred call, that panic
		// will find d in the list and will mark d._panic (this panic) aborted.
    // 保存panic指针
		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		done := true
		if d.openDefer {
			done = runOpenDeferFrame(gp, d)
			if done && !d._panic.recovered {
				addOneOpenDeferFrame(gp, 0, nil)
			}
		} else {
			p.argp = unsafe.Pointer(getargp(0))
      //执行defer
			reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
		}
		p.argp = nil

		// reflectcall did not panic. Remove d.
		if gp._defer != d {
			throw("bad defer entry in panic")
		}
		d._panic = nil

		// trigger shrinkage to test stack copy. See stack_test.go:TestStackPanic
		//GC()

		pc := d.pc
		sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
		if done {
			d.fn = nil
			gp._defer = d.link
			freedefer(d)
		}
    // recover
		if p.recovered {
			gp._panic = p.link
			if gp._panic != nil && gp._panic.goexit && gp._panic.aborted {
				// A normal recover would bypass/abort the Goexit.  Instead,
				// we return to the processing loop of the Goexit.
				gp.sigcode0 = uintptr(gp._panic.sp)
				gp.sigcode1 = uintptr(gp._panic.pc)
				mcall(recovery)
				throw("bypassed recovery failed") // mcall should not return
			}
			atomic.Xadd(&runningPanicDefers, -1)

			if done {
				// Remove any remaining non-started, open-coded
				// defer entries after a recover, since the
				// corresponding defers will be executed normally
				// (inline). Any such entry will become stale once
				// we run the corresponding defers inline and exit
				// the associated stack frame.
				d := gp._defer
				var prev *_defer
				for d != nil {
					if d.openDefer {
						if d.started {
							// This defer is started but we
							// are in the middle of a
							// defer-panic-recover inside of
							// it, so don't remove it or any
							// further defer entries
							break
						}
						if prev == nil {
							gp._defer = d.link
						} else {
							prev.link = d.link
						}
						newd := d.link
						freedefer(d)
						d = newd
					} else {
						prev = d
						d = d.link
					}
				}
			}

			gp._panic = p.link
			// Aborted panics are marked but remain on the g.panic list.
			// Remove them from the list.
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}
			if gp._panic == nil { // must be done with signal
				gp.sig = 0
			}
			// Pass information about recovering frame to recovery.
			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			mcall(recovery)
			throw("recovery failed") // mcall should not return
		}
	}

	// ran out of deferred calls - old-school panic now
	// Because it is unsafe to call arbitrary user code after freezing
	// the world, we call preprintpanics to invoke all necessary Error
	// and String methods to prepare the panic strings before startpanic.
	preprintpanics(gp._panic)
	// 奔溃打印
	fatalpanic(gp._panic) // should not return
	*(*int)(nil) = 0      // not reached
}
```



### 奔溃恢复

```go
func gorecover(argp uintptr) interface{} {
	gp := getg()
	p := gp._panic
	if p != nil && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}
```