### GC

https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/



**图 7-29 Dijkstra 插入写屏障**

因为应用程序可能包含成百上千的 Goroutine，而垃圾收集的**根对象一般包括全局变量和栈对象**，如果运行时需要在几百个 Goroutine 的栈上都开启写屏障，会带来巨大的额外开销，所以 Go 团队在实现上选择了在标记阶段完成时**暂停程序、将所有栈对象标记为灰色并重新扫描**，在活跃 Goroutine 非常多的程序中，重新扫描的过程需要占用 10 ~ 100ms 的时间



假设我们在应用程序中使用 Dijkstra 提出的插入写屏障，在一个垃圾收集器和用户程序交替运行的场景中会出现如上图所示的标记过程：

1. 垃圾收集器将根对象指向 A 对象标记成黑色并将 A 对象指向的对象 B 标记成灰色；
2. 用户程序修改 A 对象的指针，将原本指向 B 对象的指针指向 C 对象，这时触发写屏障将 C 对象标记成灰色；
3. 垃圾收集器依次遍历程序中的其他灰色对象，将它们分别标记成黑色；

Dijkstra 的插入写屏障是一种相对保守的屏障技术，它会将**有存活可能的对象都标记成灰色**以满足强三色不变性。在如上所示的垃圾收集过程中，实际上不再存活的 B 对象最后没有被回收；而如果我们在第二和第三步之间将指向 C 对象的指针改回指向 B，垃圾收集器仍然认为 C 对象是存活的，**这些被错误标记的垃圾对象只有在下一个循环才会被回收。**

插入式的 Dijkstra 写屏障虽然实现非常简单并且也能保证强三色不变性，但是它也有明显的缺点。**因为栈上的对象在垃圾收集中也会被认为是根对象**，所以为了保证内存的安全，（**因为栈上第一次扫描没开写屏障 所有要在扫描完成后STW再扫描仪器）**Dijkstra 必须为**栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象进行扫描**，这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序，垃圾收集算法的设计者需要在这两者之前做出权衡。





#### 删除写屏障

Yuasa 在 1990 年的论文 Real-time garbage collection on general-purpose machines 中提出了删除写屏障，因为一旦该写屏障开始工作，它会保证开启写屏障时堆上所有对象的可达，所以也被称作快照垃圾收集（Snapshot GC）

This guarantees that no objects will become unreachable to the garbage collector traversal all objects which are live at the beginning of garbage collection will be reached even if the pointers to them are overwritten.

```go
writePointer(slot, ptr)
    shade(*slot)
    *slot = ptr
```

上述代码会在**老对象的引用被删除时**，**将白色的老对象涂成灰色**，这样删除写屏障就可以保证弱三色不变性，老对象引用的下游对象一定可以被灰色对象引用。（**如果老对象没其他引用 需要等到下一轮gc**）

<img src="/opt/read-code/images/2021-01-02-16095599123266-yuasa-delete-write-barrier.png" alt="2021-01-02-16095599123266-yuasa-delete-write-barrier" style="zoom:50%;" />

假设我们在应用程序中使用 Yuasa 提出的删除写屏障，在一个垃圾收集器和用户程序交替运行的场景中会出现如上图所示的标记过程：

1. 垃圾收集器将根对象指向 A 对象标记成黑色并将 A 对象指向的对象 B 标记成灰色；
2. 用户程序将 A 对象原本指向 B 的指针指向 C，**触发删除写屏障，但是因为 B 对象已经是灰色的**，所以不做改变；
3. **用户程序将 B 对象原本指向 C 的指针删除，触发删除写屏障，白色的 C 对象被涂成灰色**；
4. 垃圾收集器依次遍历程序中的其他灰色对象，将它们分别标记成黑色；

上述过程中的第三步触发了 Yuasa 删除写屏障的着色，因为用户程序删除了 B 指向 C 对象的指针，所以 C 和 D 两个对象会分别违反强三色不变性和弱三色不变性：





- 强三色不变性 — **黑色的 A 对象直接指向白色的 C 对象**；
- 弱三色不变性 — 垃圾收集器无法从某个灰色对象出发，经过几个连续的白色对象访问白色的 C 和 D 两个对象；（灰色对象删除对白色对象的引用）

Yuasa 删除写屏障通过对 C 对象的着色，保证了 C 对象和下游的 D 对象能够在这一次垃圾收集的循环中存活，避免发生悬挂指针以保证用户程序的正确性。





### 增量和并发

- 增量垃圾收集 — 增量地标记和清除垃圾，降低应用程序暂停的最长时间；
- 并发垃圾收集 — 利用多核的计算资源，在用户程序执行时并发标记和清除垃圾

因为**增量和并发**两种方式都可以与用户程序交替运行，所以我们需要**使用屏障技术**保证垃圾收集的正确性；与此同时，应用程序也不能等到内存溢出时触发垃圾收集，因为当内存不足时，应用程序已经无法分配内存，这与直接暂停程序没有什么区别，增量和并发的垃圾收集需要提**前触发并在内存不足前完成整个循环**，避免程序的长时间暂停。

#### 增量收集器

增量式（Incremental）的垃圾收集是**减少程序最长暂停**时间的一种方案，它可以将**原本时间较长的暂停时间切分成多个更小的 GC 时间片**，虽然从**垃圾收集开始到结束的时间更长了**，但是这也**减少了应用程序暂停的最大时间**



需要注意的是，增量式的垃圾收集需要与三色标记法一起使用，为了保证垃圾收集的正确性，**我们需要在垃圾收集开始前打开写屏障**，这样用户程序修改内存都会先经过写屏障的处理，保证了堆内存中对象关系的强三色不变性或者弱三色不变性。虽然增量式的垃圾收集能够减少最大的程序暂停时间，但是增量式收集也会增加一次 GC 循环的总时间，在垃圾收集期间，因为写屏障的影响用户程序也需要承担额外的计算开销，所以增量式的垃圾收集也不是只带来好处的，但是总体来说还是利大于弊



#### 并发收集器

并发（Concurrent）的垃圾收集不仅**能够减少程序的最长暂停时间**，还能**减少整个垃圾收集阶段的时间**，通过开启读写屏障、**利用多核优势与用户程序并行执行**，并发垃圾收集器确实能够减少垃圾收集对应用程序的影响：

虽然并发收集器能够与用户程序一起运行，但是并不是所有阶段都可以与用户程序一起运行，**部分阶段还是需要暂停用户程序的**，不过与传统的算法相比，并发的垃圾收集可以将能够并发执行的工作尽量并发执行；当然，因为读写屏障的引入，并发的垃圾收集器也一定会带来额外开销，不仅会增加垃圾收集的总时间，还会影响用户程序，这是我们在设计垃圾收集策略时必须要注意的。





Go 语言的并发垃圾收集器会在**扫描对象之前暂停程序做一些标记对象的准备工作**，其中包括启动**后台标记的垃圾收集器以及开启写屏障**，如果在后台执行的垃圾收集器不够快，应用程序申请内存的速度超过预期，**运行时会让申请内存的应用程序辅助完成垃圾收集的扫描阶段**，在标记和标记终止阶段结束之后就会进入**异步的清理阶段，将不用的内存增量回收。**



STW 的垃圾收集器虽然需要暂停程序，但是它能够有效地控制堆内存的大小，Go 语言运行时的默认配置会在堆内存达到**上一次垃圾收集的 2 倍时**，触发新一轮的垃圾收集，这个行为可以通过环境变量 `GOGC` 调整，在默认情况下它的值为 100，即增长 100% 的堆内存才会触发 GC。



因为并发垃圾收集器会与程序一起运行，**所以它无法准确的控制堆内存的大小**，并发收集器需要在**达到目标前触发垃圾收集**，这样才能够保证内存大小的可控，并发收集器需要尽可能保证垃圾收集结束时的堆内存与用户配置的 `GOGC` 一致。





Go 语言在 v1.8 组合 Dijkstra 插入写屏障和 Yuasa 删除写屏障构成了如下所示的混合写屏障，该写屏障会**将被覆盖的对象标记成灰色并在当前栈没有扫描时将新对象也标记成灰色**：

```go
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```



为了移除栈的重扫描过程，除了引入混合写屏障之外，**在垃圾收集的标记阶段**，我们还需要**将创建的所有新对象都标记成黑色**，防止新分配的栈内存和堆内存中的对象被错误地回收，**因为栈内存在标记阶段最终都会变为黑色**，所以不再需要重新扫描栈空间。



Go 语言的垃圾收集可以分成**清除终止、标记、标记终止和清除**四个不同阶段，它们分别完成了不同的工作

1. 清理终止阶段；STW
   1. **暂停程序**，所有的处理器在这时会进入安全点（Safe point）；
   2. 如果当前垃圾收集循环是**强制触发**的，我们还需要处理还未被清理的内存管理单元；
2. 标记阶段；
   1. **将状态切换至 `_GCmark`、开启写屏障、用户程序协助（Mutator Assiste）并将根对象入队**；
   2. 恢复执行程序，**标记进程和用于协助的用户程序会开始并发标记内存中的对象**，写屏障会将被覆盖的指针和新指针都标记成灰色，**而所有新创建的对象都会被直接标记成黑色**；
   3. **开始扫描根对象**，**包括所有 Goroutine 的栈**、**全局对象以及不在堆中的运行时数据结构，扫描 Goroutine 栈期间会暂停当前处理器；**
   4. 依次处理灰色队列中的对象，将对象标记成黑色并将它们指向的对象标记成灰色；
   5. 使用分布式的终止算法检查剩余的工作，发现标记阶段完成后进入标记终止阶段；
3. 标记终止阶段； STW
   1. **暂停程序**、将状态切换至 `_GCmarktermination` **并关闭辅助标记的用户程序**；
   2. 清理处理器上的线程缓存；
4. 清理阶段；
   1. 将状态切换至 `_GCoff` 开始清理阶段，**初始化清理状态并关闭写屏障**；
   2. 恢复用户程序，所有新创建的对象会标记成白色；
   3. 后台并发清理所有的内存管理单元，当 Goroutine 申请新的内存管理单元时就会触发清理；

运行时虽然只会使用 `_GCoff`、`_GCmark` 和 `_GCmarktermination` 三个状态表示垃圾收集的全部阶段，但是在实现上却复杂很多，本节将按照垃圾收集的不同阶段详细分析其实现原理。



### 全局变量

在垃圾收集中有一些比较重要的全局变量，在分析其过程之前，我们会先逐一介绍这些重要的变量，这些变量在垃圾收集的各个阶段中会反复出现，所以理解他们的功能是非常重要的，我们先介绍一些比较简单的变量

- [`runtime.gcphase`](https://draveness.me/golang/tree/runtime.gcphase) 是垃圾收集器当前处于的阶段，可能处于 `_GCoff`、`_GCmark` 和 `_GCmarktermination`，Goroutine 在读取或者修改该阶段时需要保证原子性；
- [`runtime.gcBlackenEnabled`](https://draveness.me/golang/tree/runtime.gcBlackenEnabled) 是一个布尔值，当垃圾收集处于标记阶段时，**该变量会被置为 1**，**在这里辅助垃圾收集的用户程序和后台标记的任务可以将对象涂黑**；
- [`runtime.gcController`](https://draveness.me/golang/tree/runtime.gcController) **实现了垃圾收集的调步算法**，它能够决定触发并行垃圾收集的时间和待处理的工作；pacing
- [`runtime.gcpercent`](https://draveness.me/golang/tree/runtime.gcpercent) **是触发垃圾收集的内存增长百分比**，**默认情况下为 100**，即堆内存相比上次垃圾收集增长 100% 时应该触发 GC，并行的垃圾收集器会在到达该目标前完成垃圾收集；
- [`runtime.writeBarrier`](https://draveness.me/golang/tree/runtime.writeBarrier) 是一个包含写屏障状态的结构体，**其中的 `enabled` 字段表示写屏障的开启与关闭**；
- [`runtime.worldsema`](https://draveness.me/golang/tree/runtime.worldsema) 是全局的信号量，获取该信号量的线程有权利暂停当前应用程序；



### 触发时机

运行时会通过如下所示的 [`runtime.gcTrigger.test`](https://draveness.me/golang/tree/runtime.gcTrigger.test) 方法决定是否需要触发垃圾收集，当满足触发垃圾收集的基本条件时 — 允许垃圾收集、程序没有崩溃并且没有处于垃圾收集循环，该方法会根据三种不同方式触发进行不同的检查：

```go
func (t gcTrigger) test() bool {
	if !memstats.enablegc || panicking != 0 || gcphase != _GCoff {
		return false
	}
	switch t.kind {
    //堆内存的分配达到达控制器计算的触发堆大小
	case gcTriggerHeap:
		return memstats.heap_live >= memstats.gc_trigger
    //如果一定时间内没有触发，就会触发新的循环，该出发条件由 runtime.forcegcperiod 变量控制，默认为 2 分钟；
	case gcTriggerTime:
		if gcpercent < 0 {
			return false
		}
		lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
		return lastgc != 0 && t.now-lastgc > forcegcperiod
    //如果当前没有开启垃圾收集，则触发新的循环；
	case gcTriggerCycle:
		return int32(t.n-work.cycles) > 0
	}
	return true
}

runtime.sysmon 和 runtime.forcegchelper — 后台运行定时检查和垃圾收集；
runtime.GC — 用户程序手动触发垃圾收集；
runtime.mallocgc — 申请内存时根据堆大小触发垃圾收集；

除了使用后台运行的系统监控器和强制垃圾收集助手触发垃圾收集之外，另外两个方法会从任意处理器上触发垃圾收集，这种不需要中心组件协调的方式是在 v1.6 版本中引入的，接下来我们将展开介绍这三种不同的触发时机。
```



**后台触发**

运行时会在应用程序启动时在后台开启一个用于强制触发垃圾收集的 Goroutine，该 Goroutine 的职责非常简单 — 调用 [`runtime.gcStart`](https://draveness.me/golang/tree/runtime.gcStart) 尝试启动新一轮的垃圾收集：[`runtime.forcegchelper`](https://draveness.me/golang/tree/runtime.forcegchelper) 在大多数时间都是陷入休眠的，但是它会被系统监控器 [`runtime.sysmon`](https://draveness.me/golang/tree/runtime.sysmon) **在满足垃圾收集条件时唤醒**：

```go
func init() {
	go forcegchelper()
}

func forcegchelper() {
  //用户g
	forcegc.g = getg()
	for {
		lock(&forcegc.lock)
		atomic.Store(&forcegc.idle, 1)
		goparkunlock(&forcegc.lock, waitReasonForceGGIdle, traceEvGoBlock, 1)
		gcStart(gcTrigger{kind: gcTriggerTime, now: nanotime()})
	}
}

系统监控在每个循环中都会主动构建一个 runtime.gcTrigger 并检查垃圾收集的触发条件是否满足，如果满足条件，系统监控会将 runtime.forcegc 状态中持有的 Goroutine 加入全局队列等待调度器的调度。
func sysmon() {
	...
	for {
		...
		if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
			lock(&forcegc.lock)
			forcegc.idle = 0
			var list gList
			list.push(forcegc.g)
			injectglist(&list)
			unlock(&forcegc.lock)
		}
	}
}
```



用户程序会通过 [`runtime.GC`](https://draveness.me/golang/tree/runtime.GC) 函数在程序运行期间主动通知运行时执行，该方法在调用时会阻塞调用方直到当前垃圾收集循环完成，在垃圾收集期间也可能会通过 STW 暂停整个程序：

1. 在正式开始垃圾收集前，运行时需要通过 [`runtime.gcWaitOnMark`](https://draveness.me/golang/tree/runtime.gcWaitOnMark) 等待上一个循环的标记终止、标记和标记终止阶段完成；
2. 调用 [`runtime.gcStart`](https://draveness.me/golang/tree/runtime.gcStart) 触发新一轮的垃圾收集并通过 [`runtime.gcWaitOnMark`](https://draveness.me/golang/tree/runtime.gcWaitOnMark) 等待该轮垃圾收集的标记终止阶段正常结束；
3. 持续调用 [`runtime.sweepone`](https://draveness.me/golang/tree/runtime.sweepone) 清理全部待处理的内存管理单元并等待所有的清理工作完成，等待期间会调用 [`runtime.Gosched`](https://draveness.me/golang/tree/runtime.Gosched) 让出处理器；
4. 完成本轮垃圾收集的清理工作后，通过 [`runtime.mProf_PostSweep`](https://draveness.me/golang/tree/runtime.mProf_PostSweep) 将该阶段的堆内存状态快照发布出来，我们可以获取这时的内存状态；

```go

func GC() {
	n := atomic.Load(&work.cycles)
	gcWaitOnMark(n)
	gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})
	gcWaitOnMark(n + 1)

	for atomic.Load(&work.cycles) == n+1 && sweepone() != ^uintptr(0) {
		sweep.nbgsweep++
		Gosched()
	}

	for atomic.Load(&work.cycles) == n+1 && atomic.Load(&mheap_.sweepers) != 0 {
		Gosched()
	}

	mp := acquirem()
	cycle := atomic.Load(&work.cycles)
	if cycle == n+1 || (gcphase == _GCmark && cycle == n+2) {
		mProf_PostSweep()
	}
	releasem(mp)
}
```



#### 申请内存h

最后一个可能会触发垃圾收集的就是 [`runtime.mallocgc`](https://draveness.me/golang/tree/runtime.mallocgc) 了，我们在上一节内存分配器中曾经介绍过运行时会将堆上的对象按大小分成微对象、小对象和大对象三类，这三类对象的创建都可能会触发新的垃圾收集循环：

1. 当前线程的内存管理单元中不存在空闲空间时，创建微对象和小对象需要调用 [`runtime.mcache.nextFree`](https://draveness.me/golang/tree/runtime.mcache.nextFree) 从中心缓存或者页堆中获取新的管理单元，在这时就可能触发垃圾收集；
2. 当用户程序**申请分配 32KB 以上的大对象时**，一定会构建 [`runtime.gcTrigger`](https://draveness.me/golang/tree/runtime.gcTrigger) 结构体尝试触发垃圾收集；

通过堆内存触发垃圾收集需要比较 [`runtime.mstats`](https://draveness.me/golang/tree/runtime.mstats) 中的两个字段 — **表示垃圾收集中存活对象字节数的 `heap_live` 和表示触发标记的堆内存大小的 `gc_trigger`**；当内存中存活的对象字节数大于触发垃圾收集的堆大小时，新一轮的垃圾收集就会开始。在这里，我们将分别介绍这两个值的计算过程：

1. `heap_live` — 为了减少锁竞争，运行时只会在中心缓存分配或者释放内存管理单元以及在堆上分配大对象时才会更新；
2. `gc_trigger` — 在标记终止阶段调用 [`runtime.gcSetTriggerRatio`](https://draveness.me/golang/tree/runtime.gcSetTriggerRatio) 更新触发下一次垃圾收集的堆大小；

[`runtime.gcController`](https://draveness.me/golang/tree/runtime.gcController) 会在每个循环结束后计算触发比例并通过 [`runtime.gcSetTriggerRatio`](https://draveness.me/golang/tree/runtime.gcSetTriggerRatio) 设置 `gc_trigger`，它能够决定触发垃圾收集的时间以及用户程序和后台处理的标记任务的多少，利用反馈控制的算法根据堆的增长情况和垃圾收集 CPU 利用率确定触发垃圾收集的时机。

你可以在 [`runtime.gcControllerState.endCycle`](https://draveness.me/golang/tree/runtime.gcControllerState.endCycle) 中找到 v1.5 提出的垃圾收集调步算法[26](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#fn:26)，并在 [`runtime.gcControllerState.revise`](https://draveness.me/golang/tree/runtime.gcControllerState.revise) 中找到 v1.10 引入的软硬堆目标分离算法[27](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#fn:27)。

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	shouldhelpgc := false
	...
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			...
			v := nextFreeFast(span)
			if v == 0 {
				v, _, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			...
		} else {
			...
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
		  ...
		}
	} else {
		shouldhelpgc = true
		...
	}
	...
	if shouldhelpgc {
		if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
			gcStart(t)
		}
	}

	return x
}
```





### 垃圾收集启动

垃圾收集在启动过程一定会调用 [`runtime.gcStart`](https://draveness.me/golang/tree/runtime.gcStart)，虽然该函数的实现比较复杂，但是它的主要职责是修改全局的垃圾收集状态到 `_GCmark` 并做一些准备工作，我们会分以下几个阶段介绍该函数的实现：

1. 两次调用 [`runtime.gcTrigger.test`](https://draveness.me/golang/tree/runtime.gcTrigger.test) 检查是否满足垃圾收集条件；
2. 暂停程序、在后台启动用于处理标记任务的工作 Goroutine、确定所有内存管理单元都被清理以及其他标记阶段开始前的准备工作；
3. 进入标记阶段、准备后台的标记工作、根对象的标记工作以及微对象、恢复用户程序，进入并发扫描和标记阶段；

验证垃圾收集条件的同时，该方法还会在循环中不断调用 [`runtime.sweepone`](https://draveness.me/golang/tree/runtime.sweepone) 清理已经被标记的内存单元，完成上一个垃圾收集循环的收尾工作：



在验证了垃圾收集的条件并完成了收尾工作后，该方法会通过 `semacquire` 获取全局的 `worldsema` 信号量、调用 [`runtime.gcBgMarkStartWorkers`](https://draveness.me/golang/tree/runtime.gcBgMarkStartWorkers) **启动后台标记任务、在系统栈中调用** [`runtime.stopTheWorldWithSema`](https://draveness.me/golang/tree/runtime.stopTheWorldWithSema) 暂停程序并调用 [`runtime.finishsweep_m`](https://draveness.me/golang/tree/runtime.finishsweep_m) 保证上一个内存单元的正常回收：

```
func gcStart(trigger gcTrigger) {
	for trigger.test() && sweepone() != ^uintptr(0) {
		sweep.nbgsweep++
	}

	semacquire(&work.startSema)
	if !trigger.test() {
		semrelease(&work.startSema)
		return
	}
	...
}
```



1. 调用 [`runtime.gcBgMarkPrepare`](https://draveness.me/golang/tree/runtime.gcBgMarkPrepare) 初始化后台扫描需要的状态；
2. 调用 [`runtime.gcMarkRootPrepare`](https://draveness.me/golang/tree/runtime.gcMarkRootPrepare) 扫描栈上、全局变量等根对象并将它们加入队列；
3. 设置全局变量 [`runtime.gcBlackenEnabled`](https://draveness.me/golang/tree/runtime.gcBlackenEnabled)，用户程序和标记任务可以将对象涂黑；
4. 调用 [`runtime.startTheWorldWithSema`](https://draveness.me/golang/tree/runtime.startTheWorldWithSema) 启动程序，后台任务也会开始标记堆中的对象；

```go
func gcStart(trigger gcTrigger) {
	...
	setGCPhase(_GCmark)

	gcBgMarkPrepare()
	gcMarkRootPrepare()

	atomic.Store(&gcBlackenEnabled, 1)
	systemstack(func() {
		now = startTheWorldWithSema(trace.enabled)
		work.pauseNS += now - work.pauseStart
		work.tMark = now
	})
	semrelease(&work.startSema)
}
```







#### 暂停与恢复程序

[`runtime.stopTheWorldWithSema`](https://draveness.me/golang/tree/runtime.stopTheWorldWithSema) 和 [`runtime.startTheWorldWithSema`](https://draveness.me/golang/tree/runtime.startTheWorldWithSema) 是一对用于暂停和恢复程序的核心函数，它们有着完全相反的功能，但是程序的暂停会比恢复要复杂一些，我们来看一下前者的实现原理：



暂停程序主要使用了 [`runtime.preemptall`](https://draveness.me/golang/tree/runtime.preemptall)，该函数会调用我们在前面介绍过的 [`runtime.preemptone`](https://draveness.me/golang/tree/runtime.preemptone)，因为程序中活跃的最大处理数为 `gomaxprocs`，所以 [`runtime.stopTheWorldWithSema`](https://draveness.me/golang/tree/runtime.stopTheWorldWithSema) 在每次发现停止的处理器时都会对该变量减一，直到所有的处理器都停止运行。该函数会依次停止当前处理器、等待处于系统调用的处理器以及获取并抢占空闲的处理器，**处理器的状态在该函数返回时都会被更新至 `_Pgcstop`**，等待垃圾收集器的重新唤醒。



那现在m处于spining 没有p？

```go
func stopTheWorldWithSema() {
	_g_ := getg()
	sched.stopwait = gomaxprocs
	atomic.Store(&sched.gcwaiting, 1)
  // 对p进行抢占
	preemptall()
	_g_.m.p.ptr().status = _Pgcstop
	sched.stopwait--
  //遍历P状态 
	for _, p := range allp {
		s := p.status
		if s == _Psyscall && atomic.Cas(&p.status, s, _Pgcstop) {
			p.syscalltick++
			sched.stopwait--
		}
	}
	for {
		p := pidleget()
		if p == nil {
			break
		}
		p.status = _Pgcstop
		sched.stopwait--
	}
	wait := sched.stopwait > 0
	if wait {
		for {
			if notetsleep(&sched.stopnote, 100*1000) {
				noteclear(&sched.stopnote)
				break
			}
			preemptall()
		}
	}
}
```





程序恢复过程会使用 [`runtime.startTheWorldWithSema`](https://draveness.me/golang/tree/runtime.startTheWorldWithSema)，该函数的实现也相对比较简单：

1. 调用 [`runtime.netpoll`](https://draveness.me/golang/tree/runtime.netpoll) 从网络轮询器中获取待处理的任务并加入全局队列；
2. 调用 [`runtime.procresize`](https://draveness.me/golang/tree/runtime.procresize) 扩容或者缩容全局的处理器；
3. 调用 [`runtime.notewakeup`](https://draveness.me/golang/tree/runtime.notewakeup) 或者 [`runtime.newm`](https://draveness.me/golang/tree/runtime.newm) 依次唤醒处理器或者为处理器创建新的线程；
4. 如果当前待处理的 Goroutine 数量过多，创建额外的处理器辅助完成任务；

```go
func startTheWorldWithSema(emitTraceEvent bool) int64 {
	mp := acquirem()
	if netpollinited() {
		list := netpoll(0)
		injectglist(&list)
	}

	procs := gomaxprocs
	p1 := procresize(procs)
	sched.gcwaiting = 0
	...
	for p1 != nil {
		p := p1
		p1 = p1.link.ptr()
		if p.m != 0 {
			mp := p.m.ptr()
			p.m = 0
			mp.nextp.set(p)
      //唤醒阻塞在 stopm
			notewakeup(&mp.park)
		} else {
			newm(nil, p)
		}
	}

	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}
	...
}
```



```go
该结构体中包含大量垃圾收集的相关字段，例如：表示完成的垃圾收集循环的次数、当前循环时间和 CPU 的利用率、垃圾收集的模式等等，我们会在后面的小节中见到该结构体中的更多字段。
var work struct {
   full  lfstack          // lock-free list of full blocks workbuf
   empty lfstack          // lock-free list of empty blocks workbuf
   pad0  cpu.CacheLinePad // prevents false-sharing between full/empty and nproc/nwait

   wbufSpans struct {
      lock mutex
      // free is a list of spans dedicated to workbufs, but
      // that don't currently contain any workbufs.
      free mSpanList
      // busy is a list of all spans containing workbufs on
      // one of the workbuf lists.
      busy mSpanList
   }

   // Restore 64-bit alignment on 32-bit.
   _ uint32

   // bytesMarked is the number of bytes marked this cycle. This
   // includes bytes blackened in scanned objects, noscan objects
   // that go straight to black, and permagrey objects scanned by
   // markroot during the concurrent scan phase. This is updated
   // atomically during the cycle. Updates may be batched
   // arbitrarily, since the value is only read at the end of the
   // cycle.
   //
   // Because of benign races during marking, this number may not
   // be the exact number of marked bytes, but it should be very
   // close.
   //
   // Put this field here because it needs 64-bit atomic access
   // (and thus 8-byte alignment even on 32-bit architectures).
   bytesMarked uint64

   markrootNext uint32 // next markroot job
   markrootJobs uint32 // number of markroot jobs

   nproc  uint32
   tstart int64
   nwait  uint32
   ndone  uint32

   // Number of roots of various root types. Set by gcMarkRootPrepare.
   nFlushCacheRoots                               int
   nDataRoots, nBSSRoots, nSpanRoots, nStackRoots int

   // Each type of GC state transition is protected by a lock.
   // Since multiple threads can simultaneously detect the state
   // transition condition, any thread that detects a transition
   // condition must acquire the appropriate transition lock,
   // re-check the transition condition and return if it no
   // longer holds or perform the transition if it does.
   // Likewise, any transition must invalidate the transition
   // condition before releasing the lock. This ensures that each
   // transition is performed by exactly one thread and threads
   // that need the transition to happen block until it has
   // happened.
   //
   // startSema protects the transition from "off" to mark or
   // mark termination.
   startSema uint32
   // markDoneSema protects transitions from mark to mark termination.
   markDoneSema uint32

   bgMarkReady note   // signal background mark worker has started
   bgMarkDone  uint32 // cas to 1 when at a background mark completion point
   // Background mark completion signaling

   // mode is the concurrency mode of the current GC cycle.
   mode gcMode

   // userForced indicates the current GC cycle was forced by an
   // explicit user call.
   userForced bool

   // totaltime is the CPU nanoseconds spent in GC since the
   // program started if debug.gctrace > 0.
   totaltime int64

   // initialHeapLive is the value of memstats.heap_live at the
   // beginning of this GC cycle.
   initialHeapLive uint64

   // assistQueue is a queue of assists that are blocked because
   // there was neither enough credit to steal or enough work to
   // do.
   assistQueue struct {
      lock mutex
      q    gQueue
   }

   // sweepWaiters is a list of blocked goroutines to wake when
   // we transition from mark termination to sweep.
   sweepWaiters struct {
      lock mutex
      list gList
   }

   // cycles is the number of completed GC cycles, where a GC
   // cycle is sweep termination, mark, mark termination, and
   // sweep. This differs from memstats.numgc, which is
   // incremented at mark termination.
   cycles uint32

   // Timing/utilization stats for this cycle.
   stwprocs, maxprocs                 int32
   tSweepTerm, tMark, tMarkTerm, tEnd int64 // nanotime() of phase start

   pauseNS    int64 // total STW time this cycle
   pauseStart int64 // nanotime() of last STW

   // debug.gctrace heap sizes for this cycle.
   heap0, heap1, heap2, heapGoal uint64
}
```

```go
// gcenable is called after the bulk of the runtime initialization,
// just before we're about to start letting user code run.
// It kicks off the background sweeper goroutine, the background
// scavenger goroutine, and enables GC.
func gcenable() {
   // Kick off sweeping and scavenging.
   c := make(chan int, 2)
   go bgsweep(c)
   go bgscavenge(c)
   <-c
   <-c
   memstats.enablegc = true // now that runtime is initialized, GC is okay
}

// start forcegc helper goroutine
func init() {
	go forcegchelper()
}

func forcegchelper() {
	forcegc.g = getg()
	lockInit(&forcegc.lock, lockRankForcegc)
	for {
		lock(&forcegc.lock)
		if forcegc.idle != 0 {
			throw("forcegc: phase error")
		}
		atomic.Store(&forcegc.idle, 1)
		goparkunlock(&forcegc.lock, waitReasonForceGCIdle, traceEvGoBlock, 1)
		// this goroutine is explicitly resumed by sysmon
		if debug.gctrace > 0 {
			println("GC forced")
		}
		// Time-triggered, fully concurrent.
		gcStart(gcTrigger{kind: gcTriggerTime, now: nanotime()})
	}
}


// stopTheWorld stops all P's from executing goroutines, interrupting
// all goroutines at GC safe points and records reason as the reason
// for the stop. On return, only the current goroutine's P is running.
// stopTheWorld must not be called from a system stack and the caller
// must not hold worldsema. The caller must call startTheWorld when
// other P's should resume execution.
//
// stopTheWorld is safe for multiple goroutines to call at the
// same time. Each will execute its own stop, and the stops will
// be serialized.
//
// This is also used by routines that do stack dumps. If the system is
// in panic or being exited, this may not reliably stop all
// goroutines.
func stopTheWorld(reason string) {
	semacquire(&worldsema)
	gp := getg()
	gp.m.preemptoff = reason
	systemstack(func() {
		// Mark the goroutine which called stopTheWorld preemptible so its
		// stack may be scanned.
		// This lets a mark worker scan us while we try to stop the world
		// since otherwise we could get in a mutual preemption deadlock.
		// We must not modify anything on the G stack because a stack shrink
		// may occur. A stack shrink is otherwise OK though because in order
		// to return from this function (and to leave the system stack) we
		// must have preempted all goroutines, including any attempting
		// to scan our stack, in which case, any stack shrinking will
		// have already completed by the time we exit.
		casgstatus(gp, _Grunning, _Gwaiting)
		stopTheWorldWithSema()
		casgstatus(gp, _Gwaiting, _Grunning)
	})
}

// startTheWorld undoes the effects of stopTheWorld.
func startTheWorld() {
	systemstack(func() { startTheWorldWithSema(false) })
	// worldsema must be held over startTheWorldWithSema to ensure
	// gomaxprocs cannot change while worldsema is held.
	semrelease(&worldsema)
	getg().m.preemptoff = ""
}
```