context

https://juejin.cn/post/6844904070667321357

pprof

https://mp.weixin.qq.com/s/NHbV__IKuMr12gvfJ8RaUw

https://blog.csdn.net/qcrao/article/details/116334765



https://www.cnblogs.com/cnblogs-wangzhipeng/p/13292524.html map



抢占

https://mp.weixin.qq.com/s/EfDmwKilzVLAR-gH_7yzDQ

```
抢占发起的时机
抢占会在下列时机发生：

STW 期间
在 P 上执行 safe point 函数期间
sysmon 后台监控期间
gc pacer 分配新的 dedicated worker 期间
panic 崩溃期间
```



runtime.newstack协作式抢占

```
preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

```







### GC

#### **插入写屏障**

，每当执行类似 `*slot = ptr` 的表达式时，我们会执行上述写屏障通过 `shade` 函数尝试改变指针的颜色。**如果 `ptr` 指针是白色的，那么该函数会将该对象设置成灰色**，其他情况则保持不变。

Dijkstra 的插入写屏障是一种相对保守的屏障技术，它会将**有存活可能的对象都标记成灰色**以满足强三色不变性。在如上所示的垃圾收集过程中，实际上不再存活的 B 对象最后没有被回收；而如果我们在第二和第三步之间将指向 C 对象的指针改回指向 B，垃圾收集器仍然认为 C 对象是存活的，这些被错误标记的垃圾对象只有在下一个循环才会被回收。

插入式的 Dijkstra 写屏障虽然实现非常简单并且也能保证**强三色不变性**，但是它也有明显的缺点。因为栈上的对象在垃圾收集中也会被认为是根对象，所以为了保证内存的安全，**Dijkstra 必须为栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象进行扫描**，这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序，垃圾收集算法的设计者需要在这两者之间做出权衡。



#### 删除写屏障

老对象的**引用被删除时**，将白色的老对象涂成灰色，这样删除写屏障就可以保证弱三色不变性，老对象引用的下游对象一定可以被灰色对象引用。

Yuasa 删除写屏障通过对 C 对象的着色，保证了 C 对象和下游的 D 对象能够在**这一次垃圾收集的循环中存活**，避免发生悬挂指针以保证用户程序的正确性。



**增量垃圾收集**

 — 增量地标记和清除垃圾，降低应用程序暂停的最长时间；

需要注意的是，增量式的垃圾收集需要与三色标记法一起使用，为了保证垃圾收集的正确性，我们需要在垃圾收集开始前打开写屏障，这样用户程序修改内存都会先经过写屏障的处理，保证了堆内存中对象关系的强三色不变性或者弱三色不变性。虽然增量式的垃圾收集能够减少最大的程序暂停时间，但是增量式收集也会增加一次 GC 循环的总时间，在垃圾收集期间，因为写屏障的影响用户程序也需要承担额外的计算开销，所以增量式的垃圾收集也不是只带来好处的，但是总体来说还是利大于弊。



### 混合写屏障

该写屏障会

**将被覆盖的对象标记成灰色**

**并在当前栈没有扫描时将新对象也标记成灰色**：

在垃圾收集的标记阶段，我们还需要**将创建的所有新对象都标记成黑色**，（堆）

```go
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```





并发垃圾收集

 — 利用多核的计算资源，在用户程序执行时并发标记和清除垃圾；

并发（Concurrent）的垃圾收集不仅能够减少程序的最长暂停时间，还能减少整个垃圾收集阶段的时间，通过开启读写屏障、**利用多核优势与用户程序并行执行**，并发垃圾收集器确实能够减少垃圾收集对应用程序的影响：

虽然并发收集器能够与用户程序一起运行**，但是并不是所有阶段都可以与用户程序一起运行，部分阶段还是需要暂停用户程序的**，不过与传统的算法相比，并发的垃圾收集可以将能够并发执行的工作尽量并发执行；当然，因为读写屏障的引入，并发的垃圾收集器也一定会带来额外开销，不仅会增加垃圾收集的总时间，还会影响用户程序，这是我们在设计垃圾收集策略时必须要注意的。





### GC触发时机

- 手动GC
- Go 语言运行时的默认配置会在堆内存达到上一次垃圾收集的 2 倍时触发新一轮的垃圾收集，这个行为可以通过环境变量 `GOGC` 调整，在默认情况下它的值为 100，即增长 100% 的堆内存才会触发 GC。
- 监控线程触发
- 申请内存(当用户程序申请分配 32KB 以上的大对象时，一定会构建 [`runtime.gcTrigger`](https://draveness.me/golang/tree/runtime.gcTrigger) 结构体尝试触发垃圾收集；)

```go
func (t gcTrigger) test() bool {
	if !memstats.enablegc || panicking != 0 || gcphase != _GCoff {
		return false
	}
	switch t.kind {
	case gcTriggerHeap:
		// Non-atomic access to gcController.heapLive for performance. If
		// we are going to trigger on this, this thread just
		// atomically wrote gcController.heapLive anyway and we'll see our
		// own write.
		return gcController.heapLive >= gcController.trigger
	case gcTriggerTime:
		if gcController.gcPercent < 0 {
			return false
		}
		lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
		return lastgc != 0 && t.now-lastgc > forcegcperiod
	case gcTriggerCycle:
		// t.n > work.cycles, but accounting for wraparound.
		return int32(t.n-work.cycles) > 0
	}
	return true
}
```







1. 清理终止阶段；
   1. **暂停程序**，所有的处理器在这时会进入安全点（Safe point）；
   2. 如果当前垃圾收集循环是强制触发的，我们还需要处理还未被清理的内存管理单元；
2. 标记阶段；
   1. 将状态切换至 `_GCmark`、开启写屏障、用户程序协助（Mutator Assists）并将根对象入队；
   2. 恢复执行程序，标记进程和用于协助的用户程序会开始并发标记内存中的对象，写屏障会将被覆盖的指针和新指针都标记成灰色，而所有新创建的对象都会被直接标记成黑色；
   3. 开始扫描根对象，包括所有 Goroutine 的栈、全局对象以及不在堆中的运行时数据结构，扫描 Goroutine 栈期间会暂停当前处理器；
   4. 依次处理灰色队列中的对象，将对象标记成黑色并将它们指向的对象标记成灰色；
   5. 使用分布式的终止算法检查剩余的工作，发现标记阶段完成后进入标记终止阶段；
3. 标记终止阶段；
   1. **暂停程序**、将状态切换至 `_GCmarktermination` 并关闭辅助标记的用户程序；
   2. 清理处理器上的线程缓存；
4. 清理阶段；
   1. 将状态切换至 `_GCoff` 开始清理阶段，初始化清理状态并关闭写屏障；
   2. 恢复用户程序，所有新创建的对象会标记成白色；
   3. 后台并发清理所有的内存管理单元，当 Goroutine 申请新的内存管理单元时就会触发清理