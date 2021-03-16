https://mp.weixin.qq.com/s/01Hl_eOAP_k_YDTNFErTJQ debug

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/

```go
//example
func main() {
   ch := make(chan int)
   go func() {
      ch <- 10
   }()
   fmt.Println(<-ch)
}
```



```go
type hchan struct {
	qcount   uint           // chan中元素个数
	dataqsiz uint           // make时的size
	buf      unsafe.Pointer // buf指针
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // 发送到的位置
	recvx    uint   // 之手到的位置
	recvq    waitq  // 接收消息等待队列
	sendq    waitq  // 发送消息等待队列
	// with stack shrinking.
	lock mutex
}
func makechan(t *chantype, size int) *hchan {
   elem := t.elem
   if elem.size >= 1<<16 {
      throw("makechan: invalid channel element type")
   }
   if hchanSize%maxAlign != 0 || elem.align > maxAlign {
      throw("makechan: bad alignment")
   }
   //计算内存
   mem, overflow := math.MulUintptr(elem.size, uintptr(size))
   if overflow || mem > maxAlloc-hchanSize || size < 0 {
      panic(plainError("makechan: size out of range"))
   }
   // 当Hchan buf没有包含指针，gc不会扫描
   // buf指向相同的分配，elemtype是持久化的
   // SudoG再他们自己的线程引用，因此可以被收集
   var c *hchan
   switch {
   case mem == 0:
	   //无缓冲
       c = (*hchan)(mallocgc(hchanSize, nil, true))
      // Race detector用来检测同步
      c.buf = c.raceaddr()
   case elem.ptrdata == 0:
       // Elements不包含指针
       // 分配hcan和buf
      c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
      c.buf = add(unsafe.Pointer(c), hchanSize)
   default:
      //包含指针
      c = new(hchan)
      c.buf = mallocgc(mem, elem, true)
   }
   c.elemsize = uint16(elem.size)
   c.elemtype = elem
   c.dataqsiz = uint(size)
   lockInit(&c.lock, lockRankHchan)
   return c
}


// entry point for c <- x from compiled code
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
    //getcallerpc 调用者pc 计数器
	chansend(c, elem, true, getcallerpc())
}

// 通用单通道发送/接收，如果block不是nil，
// 睡眠可以唤醒当g.param==nil 当一个channel陷入睡眠的通道已经关闭，它容易循环并且重新操作，我们将看到它现在关闭
// select下有有default block false

如果chan未初始化非select default的情况下阻塞
如果非阻塞并且未关闭队列满了 返回 
如果关闭了panic

判断接收队列是否为空，如果不为空说明队列内容为空的直接复制内容到消费者栈上，并唤返回
如果缓冲队列还有空间写入缓冲队列 返回

如果非阻塞的返回

把当前g挂起放入发送队列中 等待唤醒

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 未初始化的chan
    if c == nil {
        //select下
		if !block {
			return false
		}
        // 挂起
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
    // 如果是非阻塞的并且未关闭队列已经满
    // full 如果非阻塞队列判断 有没有等待的消费者，否则判断qcount==datasiz	
	if !block && c.closed == 0 && full(c) {
		return false
	}
	lock(&c.lock)
	// 如果已经关闭 panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
	//如果接收队列不为空 说明缓冲队列为0 直接数据复制到等待的goroutine
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	//如果队列还没满
	if c.qcount < c.dataqsiz {
        // 计算出可用的位置
		qp := chanbuf(c, c.sendx)
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
	// 如果已经满了 并且非阻塞 返回
	if !block {
		unlock(&c.lock)
		return false
	}
	// 挂起当前的goroutine 等接收者来释放
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
    // 挂起
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
    //gp.param 是消费者goread前设置
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	mysg.c = nil
    //释放sudog
	releaseSudog(mysg)
	return true
}

// send处理一个send操作再一个空的channel c
// 值ep发送直接复制到接收者sg
// 接收者将被唤醒
// channel c必须为空并且被锁，send完之后解锁
// sg 必须退出队列
// ep必须非nil并指向堆或者调用者的栈
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	//直接拷贝到接收的g的接收变量
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
    //唤醒
	goready(gp, skip+1)
}


// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

// chanrecv从channel c接收值并写入到ep
// ep可能是空当接收的值被忽略时
// 如果block == false并且没有数据可取，返回(false false)
// 否则如果chan已经关闭，空的*ep返回（true，false）
// 否则填充*ep并返回(true,true)
// 一个非空的ep必须指向堆或者调用者的栈

如果chan为nil阻塞挂起，非阻塞的返回
如果非阻塞队列为空
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 未初始化
    if c == nil {
        // 非阻塞 返回
		if !block {
			return
		}
        //挂起
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
	// 如果非阻塞 chan缓冲空
    empty 如果是阻塞的判断是否有发送者
    非阻塞的判断qcount
	if !block && empty(c) {
		//如果未关闭
		if atomic.Load(&c.closed) == 0 {
			return
		}
		// 如果已经关闭 chan缓冲空 ep不为nil
 		if empty(c) {
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}
	lock(&c.lock)
    //如果已经关闭 缓冲无数据
	if c.closed != 0 && c.qcount == 0 {
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}
	// 有等待发送的队列 说明没有接收者
	if sg := c.sendq.dequeue(); sg != nil {
        //找到等待发送的sender，如果是无缓冲队列的直接从发送者拷贝数据
        //如果是有缓冲队列 从对头接收值并将sender的值copy到缓冲区尾部
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	//没有等待的sender 从缓冲数据读取
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
	// 无等待发送的 无缓冲数据
	if !block {
		unlock(&c.lock)
		return false, false
	}

	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
    //挂起
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
    // 如果关闭了
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
    //释放sudog
	releaseSudog(mysg)
	return true, !closed
}


// recv处理一个receive操作再一个满的channel
// 发送方的值放入到channel唤醒sender
// 值被当前接收者写入到ep
// 对于同步channel，两个值是一样的
// 对于异步channel，接收者从channel的buffer那道sender写入channel的buffer的值
// channel c必须满队列并且锁住，recv解锁
// sg必须从队列脱离
// 一个非空的ep必须指向堆或者调用者的栈
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	//无缓冲队列直接复制
    if c.dataqsiz == 0 {
		if ep != nil {
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
        // 缓冲队列已经满了 拿head的元素
        // 新的元素差如入到队尾
		qp := chanbuf(c, c.recvx)
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
    // 唤醒sender
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1)
}

关闭一个nil 的panic
重复关闭 panic
找到所有的等待接收的g 释放
找到所有等待发送的g 释放 因为已关闭 gp.parm==nil 唤醒的g会panic
func closechan(c *hchan) {
    //未初始化
	if c == nil {
		panic(plainError("close of nil channel"))
	}
    // 已经关闭
	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}
	c.closed = 1
	var glist gList
	// 释放所有的reader
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		gp := sg.g
		gp.param = nil
		glist.push(gp)
	}
	//释放所有的sender将会panic
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		gp := sg.g
        // 唤醒的时候发现param为nil panic
		gp.param = nil
		glist.push(gp)
	}
	unlock(&c.lock)
	// 唤醒
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}

```



### select

https://blog.csdn.net/qq_25870633/article/details/83339538

https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/

```go
func main() {
   var ch chan int = make(chan int, 1)
   select {
   case v, ok := <-ch:
      fmt.Println(v, ok)
   default:
      fmt.Println("default")
   }
}

//无case
func block() {
	gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1) // forever
}

//单个chan+default
// compiler implements
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}
//单个chan+default
// compiler implements
//
//	select {
//	case v = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbrecv(&v, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

// Select case descriptor.
// Known to compiler.
// Changes here must also be made in src/cmd/internal/gc/select.go's scasetype.
type scase struct {
	c           *hchan         // chan
	elem        unsafe.Pointer // data element
	kind        uint16
	pc          uintptr // race pc (for race detector / msan)
	releasetime int64
}

// selectgo implements the select statement.
//
// cas0 points to an array of type [ncases]scase, and order0 points to
// an array of type [2*ncases]uint16 where ncases must be <= 65536.
// Both reside on the goroutine's stack (regardless of any escaping in
// selectgo).
//
// selectgo returns the index of the chosen scase, which matches the
// ordinal position of its respective select{recv,send,default} call.
// Also, if the chosen scase was a receive operation, it reports whether
// a value was received.
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	// NOTE: In order to maintain a lean stack size, the number of scases
	// is capped at 65536.
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]

	// Replace send/receive cases involving nil channels with
	// caseNil so logic below can assume non-nil channel.
	for i := range scases {
		cas := &scases[i]
		if cas.c == nil && cas.kind != caseDefault {
			*cas = scase{}
		}
	}

	// The compiler rewrites selects that statically have
	// only 0 or 1 cases plus default into simpler constructs.
	// The only way we can end up with such small sel.ncase
	// values here is for a larger select in which most channels
	// have been nilled out. The general code handles those
	// cases correctly, and they are rare enough not to bother
	// optimizing (and needing to test).

	// generate permuted order
	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[i] = pollorder[j]
		pollorder[j] = uint16(i)
	}

	// sort the cases by Hchan address to get the locking order.
	// simple heap sort, to guarantee n log n time and constant stack footprint.
	for i := 0; i < ncases; i++ {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	for i := ncases - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}

	// lock all the channels involved in the select
	sellock(scases, lockorder)

	var (
		gp     *g
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

loop:
	// pass 1 - look for something already waiting
	var dfli int
	var dfl *scase
	var casi int
	var cas *scase
	var recvOK bool
	for i := 0; i < ncases; i++ {
		casi = int(pollorder[i])
		cas = &scases[casi]
		c = cas.c

		switch cas.kind {
		case caseNil:
			continue

		case caseRecv:
			sg = c.sendq.dequeue()
			if sg != nil {
				goto recv
			}
			if c.qcount > 0 {
				goto bufrecv
			}
			if c.closed != 0 {
				goto rclose
			}

		case caseSend:
			if raceenabled {
				racereadpc(c.raceaddr(), cas.pc, chansendpc)
			}
			if c.closed != 0 {
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			if c.qcount < c.dataqsiz {
				goto bufsend
			}

		case caseDefault:
			dfli = casi
			dfl = cas
		}
	}

	if dfl != nil {
		selunlock(scases, lockorder)
		casi = dfli
		cas = dfl
		goto retc
	}

	// pass 2 - enqueue on all chans
	gp = getg()
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		if cas.kind == caseNil {
			continue
		}
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		// No stack splits between assigning elem and enqueuing
		// sg on gp.waiting where copystack can find it.
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// Construct waiting list in lock order.
		*nextp = sg
		nextp = &sg.waitlink

		switch cas.kind {
		case caseRecv:
			c.recvq.enqueue(sg)

		case caseSend:
			c.sendq.enqueue(sg)
		}
	}

	// wait for someone to wake us up
	gp.param = nil
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)
	gp.activeStackChans = false

	sellock(scases, lockorder)

	gp.selectDone = 0
	sg = (*sudog)(gp.param)
	gp.param = nil

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
	casi = -1
	cas = nil
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	for _, casei := range lockorder {
		k = &scases[casei]
		if k.kind == caseNil {
			continue
		}
		if sglist.releasetime > 0 {
			k.releasetime = sglist.releasetime
		}
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			casi = int(casei)
			cas = k
		} else {
			c = k.c
			if k.kind == caseSend {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	if cas == nil {
		// We can wake up with gp.param == nil (so cas == nil)
		// when a channel involved in the select has been closed.
		// It is easiest to loop and re-run the operation;
		// we'll see that it's now closed.
		// Maybe some day we can signal the close explicitly,
		// but we'd have to distinguish close-on-reader from close-on-writer.
		// It's easiest not to duplicate the code and just recheck above.
		// We know that something closed, and things never un-close,
		// so we won't block again.
		goto loop
	}

	c = cas.c

	if cas.kind == caseRecv {
		recvOK = true
	}
	selunlock(scases, lockorder)
	goto retc

bufrecv:
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc

bufsend:
	// can send to buffer
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc

recv:
	// can receive from sleeping sender (sg)
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	recvOK = true
	goto retc

rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	goto retc

send:
	// can send to a sleeping receiver (sg)
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc

retc:

	return casi, recvOK

sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```













```go
// Puts the current goroutine into a waiting state and calls unlockf.
// If unlockf returns false, the goroutine is resumed.
// unlockf must not access this G's stack, as it may be moved between
// the call to gopark and the call to unlockf.
// Reason explains why the goroutine has been parked.
// It is displayed in stack traces and heap dumps.
// Reasons should be unique and descriptive.
// Do not re-use reasons, add new ones.
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
   if reason != waitReasonSleep {
      checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
   }
   mp := acquirem()
   gp := mp.curg
   status := readgstatus(gp)
   if status != _Grunning && status != _Gscanrunning {
      throw("gopark: bad g status")
   }
   mp.waitlock = lock
   mp.waitunlockf = unlockf
   gp.waitreason = reason
   mp.waittraceev = traceEv
   mp.waittraceskip = traceskip
   releasem(mp)
   // can't do anything that might move the G between Ms here.
    // 切换到g0执行park_m
   mcall(park_m)
}

park_m首先把当前goroutine的状态设置为_Gwaiting（因为它正在等待其它goroutine往channel里面写数据），然后调用dropg函数解除g和m之间的关系，最后通过调用schedule函数进入调度循环，schedule函数我们也详细分析过，它首先会从运行队列中挑选出一个goroutine，然后调用gogo函数切换到被挑选出来的goroutine去运行。因为main goroutine在读取channel被阻塞之前已经把创建好的g2放入了运行队列，所以在这里schedule会把g2调度起来运行，这里完成了一次从main goroutine到g2调度（我们假设只有一个工作线程在进行调度）。



func park_m(gp *g) {
    //g0
	_g_ := getg()
	//阻塞的g状态变为waiting	
	casgstatus(gp, _Grunning, _Gwaiting)
    //解除g和m关系
	dropg()

	if fn := _g_.m.waitunlockf; fn != nil {
		ok := fn(gp, _g_.m.waitlock)
		_g_.m.waitunlockf = nil
		_g_.m.waitlock = nil
        //执行失败 状态变更回来 继续执行g (那些场景)?
		if !ok {
			casgstatus(gp, _Gwaiting, _Grunnable)
			execute(gp, true) // Schedule it back, never returns.
		}
	}
    //调度
	schedule()
}


func goready(gp *g, traceskip int) {
	systemstack(func() {
		ready(gp, traceskip, true)
	})
}

// Mark gp ready to run.
func ready(gp *g, traceskip int, next bool) {
	
	status := readgstatus(gp)
	
	// Mark runnable.
	_g_ := getg() //g0
	mp := acquirem() // disable preemption because it can be holding p in a local var
	if status&^_Gscan != _Gwaiting {
		dumpgstatus(gp)
		throw("bad g->status in ready")
	}

	// status is Gwaiting or Gscanwaiting, make Grunnable and put on runq
    // 状态变为待运行
	casgstatus(gp, _Gwaiting, _Grunnable)
    //放到待运行队列
	runqput(_g_.m.p.ptr(), gp, next)
    //有空闲的p而且没有正在偷取goroutine的工作线程，则需要唤醒p出来工作
	wakep()
	releasem(mp)
}
```