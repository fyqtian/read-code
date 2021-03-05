### 内存分配



https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/

https://louyuting.blog.csdn.net/article/details/102945046



### 分配方法 

编程语言的内存分配器一般包含两种分配方法，一种**线性分配器**（Sequential Allocator，Bump Allocator），另一种是**空闲链表分配器**（Free-List Allocator），这两种分配方法有着不同的实现机制和特性，本节会依次介绍它们的分配过程。



线性分配（Bump Allocator）是一种高效的内存分配方法，但是有较大的局限性。当我们使用线性分配器时，只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置，

![](..\images\2020-02-29-15829868066435-bump-allocator.png)

虽然线性分配器实现为它带来了较快的执行速度以及较低的实现复杂度，但是线性分配器无法在内存被释放时重用内存。如下图所示，如果已经分配的内存被回收，线性分配器无法重新利用红色的内存

![](D:\gocode\read-code\images\2020-02-29-15829868066441-bump-allocator-reclaim-memory.png)

因为线性分配器具有上述特性，所以需要与合适的垃圾回收算法配合使用，例如：**标记压缩（Mark-Compact）、复制回收（Copying GC）和分代回收（Generational GC）等算法，它们可以通过*拷贝的方式整理存活*对象的碎片，将空闲内存定期合并，这样就能利用线性分配器的效率提升内存分配器的性能了。**

因为线性分配器需要与具有拷贝特性的垃圾回收算法配合，所以 C 和 C++ 等需要直接对外暴露指针的语言就无法使用该策略，我们会在下一节详细介绍常见垃圾回收算法的设计原理。



#### 空闲链表分配器 

空闲链表分配器（Free-List Allocator）可以**重用已经被释放的内存**，它在内部会维护一个类似**链表的数据结构**。当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表：

![](..\images\2020-02-29-15829868066446-free-list-allocator.png)

因为不同的内存块通过指针构成了链表，所以使用这种方式的分配器可以**重新利用回收的资源**，但是因为分配内存时**需要遍历链表，所以它的时间复杂度是 O(n))**。空闲链表分配器可以选择不同的策略在链表中的内存块中进行选择，最常见的是以下四种：

- 首次适应（First-Fit）— 从链表头开始遍历，选择第一个大小大于申请内存的内存块；
- 循环首次适应（Next-Fit）— 从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块；
- 最优适应（Best-Fit）— 从链表头遍历整个链表，选择最合适的内存块；
- 隔离适应（Segregated-Fit）— 将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块；

上述四种策略的前三种就不过多介绍了，Go 语言使用的**内存分配策略与第四种策略有些相似**，我们通过下图了解该策略的原理：

![](D:\gocode\read-code\images\2020-02-29-15829868066452-segregated-list.png)



如上图所示，该策略会将内存分割成由 **4、8、16、32 字节的内存块组成的链表**，当我们向内存分配器申请 8 字节的内存时，它会在上图中找到满足条件的空闲内存块并返回。隔离适应的分配策略减少了**需要遍历的内存块数量**，提高了**内存分配的效率**。



线程缓存分配（Thread-Caching Malloc，TCMalloc）是用于分配内存的机制，它比 glibc 中的 `malloc` 还要快很多[2](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#fn:2)。Go 语言的内存分配器就借鉴了 TCMalloc 的设计实现高速的内存分配，**它的核心理念是使用多级缓存将对象根据大小分类**，并按照类别实施不同的分配策略。

#### 对象大小 [#](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#对象大小)

Go 语言的内存分配器会根据申请**分配的内存大小选择不同的处理逻辑**，运行时根据对象的大小将对象分成微对象、小对象和大对象三种：

|  类别  |     大小      |
| :----: | :-----------: |
| 微对象 |  `(0, 16B)`   |
| 小对象 | `[16B, 32KB]` |
| 大对象 | `(32KB, +∞)`  |

内存分配器不仅会区别对待大小不同的对象，**还会将内存分成不同的级别分别管理**，TCMalloc 和 Go 运行时分配器都会引入线程缓存（Thread Cache）、中心缓存（Central Cache）和页堆（Page Heap）三个组件分级管理内存：

<img src="..\images\2020-02-29-15829868066457-multi-level-cache.png" style="zoom:50%;" />

**线程缓存属于每一个独立的线程**，它能够满足线程上绝大多数的内存分配需求，因为不涉及多线程，所以也不需要使用互斥锁来保护内存，这能够减少锁竞争带来的性能损耗。当线程缓存不能满足需求时，**运行时会使用中心缓存作为补充解决小对象的内存分配**，在遇到 32KB 以上的对象时，**内存分配器会选择页堆直接分配大内存**。

这种多层级的内存分配设计与计算机操作系统中的多级缓存有些类似，因为多数的对象都是小对象，我们可以通过线程缓存和中心缓存提供足够的内存空间，发现资源不足时从上一级组件中获取更多的内存资源。



#### 线性内存

Go 语言程序的 1.10 版本在启动时会**初始化整片虚拟内存区域**，如下所示的三个区域 `spans`、`bitmap` 和 `arena` 分别预留了 512MB、16GB 以及 512GB 的内存空间，这些内存并不是真正存在的物理内存，而是虚拟内存：

<img src="..\images\heap-before-go-1-10.png" style="zoom:50%;" />

- `spans` 区域存储了指向内存管理单元 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan) 的指针，每个内存单元会管理几页的内存空间，每页大小为 8KB；
- `bitmap` **用于标识 `arena` 区域中的那些地址保存了对象**，位图中的每个字节都会表示堆区中的 32 字节是否包含空闲；
- `arena` 区域是真正的堆区，运行时会将 8KB 看做一页，这些内存页中存储了所有在堆上初始化的对象；

对于任意一个地址，**我们都可以根据 `arena` 的基地址计算该地址所在的页数并通过 `spans` 数组获得管理该片内存的管理单元 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan)**，`spans` 数组中多个连续的位置可能对应同一个 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan) 结构。

Go 语言在**垃圾回收时会根据指针的地址判断对象是否在堆中**，并通过上一段中介绍的过程找到**管理该对象的 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan)**。这些都建立在**堆区的内存是连续的**这一假设上。这种设计虽然简单并且方便，但是在 C 和 Go 混合使用时会导致程序崩溃：

1. 分配的内存地址会发生冲突，导致堆的初始化和扩容失败；
2. 没有被预留的大块内存可能会被分配给 C 语言的二进制，导致扩容后的堆不连续；



### 地址空间 

因为所有的内存最终都是要从操作系统中申请的，所以 Go 语言的运行时构建了操作系统的内存管理抽象层，该抽象层将运行时管理的地址空间分成以下四种状态

|    状态    |                             解释                             |
| :--------: | :----------------------------------------------------------: |
|   `None`   |         内存没有被保留或者映射，是地址空间的默认状态         |
| `Reserved` |        运行时持有该地址空间，但是访问该内存会导致错误        |
| `Prepared` | 内存被保留，一般没有对应的物理内存访问该片内存的行为是未定义的可以快速转换到 `Ready` 状态 |
|  `Ready`   |                        可以被安全访问                        |



### 稀疏内存

<img src="..\images\2020-02-29-15829868066479-go-memory-layout.png" style="zoom:50%;" />



所有的 Go 语言程序都会在**启动时初始化如上图所示的内存布局**，每一个处理器都会分配一个线程缓存 [`runtime.mcache`](https://draveness.me/golang/tree/runtime.mcache) 用于处理微对象和小对象的分配，它们会持有内存管理单元 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan)。mcache挂在p上

每个类型的**内存管理单元都会管理特定大小的对象**，当内存管理单元中不存在空闲对象时，它们会从 [`runtime.mheap`](https://draveness.me/golang/tree/runtime.mheap) 持有的 134 个中心缓存 [`runtime.mcentral`](https://draveness.me/golang/tree/runtime.mcentral) 中获取新的内存单元，中心缓存属于全局的堆结构体 [`runtime.mheap`](https://draveness.me/golang/tree/runtime.mheap)，它会从操作系统中申请内存。

在 amd64 的 Linux 操作系统上，[`runtime.mheap`](https://draveness.me/golang/tree/runtime.mheap) 会持有 4,194,304 [`runtime.heapArena`](https://draveness.me/golang/tree/runtime.heapArena)，每个 [`runtime.heapArena`](https://draveness.me/golang/tree/runtime.heapArena) 都会管理 64MB 的内存，单个 Go 语言程序的内存上限也就是 256TB。





```go
p申请独立的缓存
pp.mcache = allocmcache()
func allocmcache() *mcache {
	var c *mcache
	systemstack(func() {
		lock(&mheap_.lock)
		c = (*mcache)(mheap_.cachealloc.alloc())
		c.flushGen = mheap_.sweepgen
		unlock(&mheap_.lock)
	})
	for i := range c.alloc {
        初始化后的都是空的占位符 emptymspan
		c.alloc[i] = &emptymspan
	}
	c.nextSample = nextSample()
	return c
}

refill会为线程缓存获取一个指定跨度类的内存管理单元，被替换的单元不能包含空闲的内存空间，而获取的单元中需要至少包含一个空闲对象用于分配内存：
func (c *mcache) refill(spc spanClass) {
	s := c.alloc[spc]
	s = mheap_.central[spc].mcentral.cacheSpan()
	c.alloc[spc] = s
}

type mcache struct {
	tiny             uintptr
	tinyoffset       uintptr
	local_tinyallocs uintptr
}
微分配器只会用于分配非指针类型的内存，上述三个字段中 tiny 会指向堆中的一片内存，tinyOffset 是下一个空闲内存所在的偏移量，最后的 local_tinyallocs 会记录内存分配器中分配的对象个数。



中心缓存

runtime.mcentral 是内存分配器的中心缓存，与线程缓存不同，访问中心缓存中的内存管理单元需要使用互斥锁：

每个中心缓存都会管理某个跨度类的内存管理单元，它会同时持有两个 runtime.spanSet，分别存储包含空闲对象和不包含空闲对象的内存管理单元。
type mcentral struct {
	spanclass spanClass
	partial  [2]spanSet
	full     [2]spanSet
}

```



线程缓存会通过中心缓存的 [`runtime.mcentral.cacheSpan`](https://draveness.me/golang/tree/runtime.mcentral.cacheSpan) 方法获取新的内存管理单元，该方法的实现比较复杂，我们可以将其分成以下几个部分：

1. 调用 [`runtime.mcentral.partialSwept`](https://draveness.me/golang/tree/runtime.mcentral.partialSwept) 从清理过的、包含空闲空间的 [`runtime.spanSet`](https://draveness.me/golang/tree/runtime.spanSet) 结构中查找可以使用的内存管理单元；
2. 调用 [`runtime.mcentral.partialUnswept`](https://draveness.me/golang/tree/runtime.mcentral.partialUnswept) 从未被清理过的、有空闲对象的 [`runtime.spanSet`](https://draveness.me/golang/tree/runtime.spanSet) 结构中查找可以使用的内存管理单元；
3. 调用 [`runtime.mcentral.fullUnswept`](https://draveness.me/golang/tree/runtime.mcentral.fullUnswept) 获取未被清理的、不包含空闲空间的 [`runtime.spanSet`](https://draveness.me/golang/tree/runtime.spanSet) 中获取内存管理单元并通过 [`runtime.mspan.sweep`](https://draveness.me/golang/tree/runtime.mspan.sweep) 清理它的内存空间；
4. 调用 [`runtime.mcentral.grow`](https://draveness.me/golang/tree/runtime.mcentral.grow) 从堆中申请新的内存管理单元；
5. 更新内存管理单元的 `allocCache` 等字段帮助快速分配内存；

首先我们会在中心缓存的空闲集合中查找可用的 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan)，运行时总是会先从获取清理过的内存管理单元，后检查未清理的内存管理单元：

```go
// Allocate a span to use in an mcache.
func (c *mcentral) cacheSpan() *mspan {
   if !go115NewMCentralImpl {
      return c.oldCacheSpan()
   }
   // Deduct credit for this span allocation and sweep if necessary.
   spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
   deductSweepCredit(spanBytes, 0)

   sg := mheap_.sweepgen

   traceDone := false
   if trace.enabled {
      traceGCSweepStart()
   }

   // If we sweep spanBudget spans without finding any free
   // space, just allocate a fresh span. This limits the amount
   // of time we can spend trying to find free space and
   // amortizes the cost of small object sweeping over the
   // benefit of having a full free span to allocate from. By
   // setting this to 100, we limit the space overhead to 1%.
   //
   // TODO(austin,mknyszek): This still has bad worst-case
   // throughput. For example, this could find just one free slot
   // on the 100th swept span. That limits allocation latency, but
   // still has very poor throughput. We could instead keep a
   // running free-to-used budget and switch to fresh span
   // allocation if the budget runs low.
   spanBudget := 100

   var s *mspan

   // Try partial swept spans first.
   if s = c.partialSwept(sg).pop(); s != nil {
      goto havespan
   }

   // Now try partial unswept spans.
   for ; spanBudget >= 0; spanBudget-- {
      s = c.partialUnswept(sg).pop()
      if s == nil {
         break
      }
      if atomic.Load(&s.sweepgen) == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
         // We got ownership of the span, so let's sweep it and use it.
         s.sweep(true)
         goto havespan
      }
      // We failed to get ownership of the span, which means it's being or
      // has been swept by an asynchronous sweeper that just couldn't remove it
      // from the unswept list. That sweeper took ownership of the span and
      // responsibility for either freeing it to the heap or putting it on the
      // right swept list. Either way, we should just ignore it (and it's unsafe
      // for us to do anything else).
   }
   // Now try full unswept spans, sweeping them and putting them into the
   // right list if we fail to get a span.
   for ; spanBudget >= 0; spanBudget-- {
      s = c.fullUnswept(sg).pop()
      if s == nil {
         break
      }
      if atomic.Load(&s.sweepgen) == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
         // We got ownership of the span, so let's sweep it.
         s.sweep(true)
         // Check if there's any free space.
         freeIndex := s.nextFreeIndex()
         if freeIndex != s.nelems {
            s.freeindex = freeIndex
            goto havespan
         }
         // Add it to the swept list, because sweeping didn't give us any free space.
         c.fullSwept(sg).push(s)
      }
      // See comment for partial unswept spans.
   }
   if trace.enabled {
      traceGCSweepDone()
      traceDone = true
   }

   // We failed to get a span from the mcentral so get one from mheap.
   s = c.grow()
   if s == nil {
      return nil
   }

   // At this point s is a span that should have free slots.
havespan:
   if trace.enabled && !traceDone {
      traceGCSweepDone()
   }
   n := int(s.nelems) - int(s.allocCount)
   if n == 0 || s.freeindex == s.nelems || uintptr(s.allocCount) == s.nelems {
      throw("span has no free objects")
   }
   // Assume all objects from this span will be allocated in the
   // mcache. If it gets uncached, we'll adjust this.
   atomic.Xadd64(&c.nmalloc, int64(n))
   usedBytes := uintptr(s.allocCount) * s.elemsize
   atomic.Xadd64(&memstats.heap_live, int64(spanBytes)-int64(usedBytes))
   if trace.enabled {
      // heap_live changed.
      traceHeapAlloc()
   }
   if gcBlackenEnabled != 0 {
      // heap_live changed.
      gcController.revise()
   }
   freeByteBase := s.freeindex &^ (64 - 1)
   whichByte := freeByteBase / 8
   // Init alloc bits cache.
   s.refillAllocCache(whichByte)

   // Adjust the allocCache so that s.freeindex corresponds to the low bit in
   // s.allocCache.
   s.allocCache >>= s.freeindex % 64

   return s
}
```







**内存分配**

```go
// implementation of new builtin
// compiler (both frontend and SSA backend) knows the signature
// of this function
func newobject(typ *_type) unsafe.Pointer {
   return mallocgc(typ.size, typ, true)
}

// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if size == 0 {
		return unsafe.Pointer(&zerobase)
	}
	// assistG is the G to charge for this allocation, or nil if
	// GC is not currently active.
    // 辅助G负责这次分配,如果gc不在运行assistG==nil 
	var assistG *g
	if gcBlackenEnabled != 0 {
		// Charge the current user G for this allocation.
		assistG = getg()
		if assistG.m.curg != nil {
			assistG = assistG.m.curg
		}
		// Charge the allocation against the G. We'll account
		// for internal fragmentation at the end of mallocgc.
		assistG.gcAssistBytes -= int64(size)

		if assistG.gcAssistBytes < 0 {
			// This G is in debt. Assist the GC to correct
			// this before allocating. This must happen
			// before disabling preemption.
			gcAssistAlloc(assistG)
		}
	}

	// Set mp.mallocing to keep from being preempted by GC.
	mp := acquirem()
	mp.mallocing = 1

	shouldhelpgc := false
	dataSize := size
	var c *mcache
	if mp.p != 0 {
		c = mp.p.ptr().mcache
	} else {
		// We will be called without a P while bootstrapping,
		// in which case we use mcache0, which is set in mallocinit.
		// mcache0 is cleared when bootstrapping is complete,
		// by procresize.
		c = mcache0
		if c == nil {
			throw("malloc called with no P")
		}
	}
	var span *mspan
	var x unsafe.Pointer
    // Nil或者非指针
	noscan := typ == nil || typ.ptrdata == 0
    // maxSmallSize = 32k
	if size <= maxSmallSize {
        // 微对象分配 maxTinySize = 16
		if noscan && size < maxTinySize {
			// Tiny allocator.
			//
			// Tiny allocator combines several tiny allocation requests
			// into a single memory block. The resulting memory block
			// is freed when all subobjects are unreachable. The subobjects
			// must be noscan (don't have pointers), this ensures that
			// the amount of potentially wasted memory is bounded.
			//
			// Size of the memory block used for combining (maxTinySize) is tunable.
			// Current setting is 16 bytes, which relates to 2x worst case memory
			// wastage (when all but one subobjects are unreachable).
			// 8 bytes would result in no wastage at all, but provides less
			// opportunities for combining.
			// 32 bytes provides more opportunities for combining,
			// but can lead to 4x worst case wastage.
			// The best case winning is 8x regardless of block size.
			//
			// Objects obtained from tiny allocator must not be freed explicitly.
			// So when an object will be freed explicitly, we ensure that
			// its size >= maxTinySize.
			//
			// SetFinalizer has a special case for objects potentially coming
			// from tiny allocator, it such case it allows to set finalizers
			// for an inner byte of a memory block.
			//
			// The main targets of tiny allocator are small strings and
			// standalone escaping variables. On a json benchmark
			// the allocator reduces number of allocations by ~12% and
			// reduces heap size by ~20%.
			off := c.tinyoffset
			// Align tiny pointer for required (conservative) alignment.
            // 对齐
			if size&7 == 0 {
				off = alignUp(off, 8)
			} else if size&3 == 0 {
				off = alignUp(off, 4)
			} else if size&1 == 0 {
				off = alignUp(off, 2)
			}
            //填充完以后还是小于<16b 并且mcache.tiny不为0
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
            // 分配一个新的微小块
			span = c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if size < c.tinyoffset || c.tiny == 0 {
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
            //分配16b-32k的对象
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
			} else {
				sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			span = c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
        //大内存 堆上直接分配
		shouldhelpgc = true
		systemstack(func() {
			span = largeAlloc(size, needzero, noscan)
		})
		span.freeindex = 1
		span.allocCount = 1
		x = unsafe.Pointer(span.base())
		size = span.elemsize
	}

	var scanSize uintptr
    //如果需要扫描
	if !noscan {
		// If allocating a defer+arg block, now that we've picked a malloc size
		// large enough to hold everything, cut the "asked for" size down to
		// just the defer header, so that the GC bitmap will record the arg block
		// as containing nothing at all (as if it were unused space at the end of
		// a malloc block caused by size rounding).
		// The defer arg areas are scanned as part of scanstack.
        
        //如果是defer 对arg做标记
		if typ == deferType {
			dataSize = unsafe.Sizeof(_defer{})
		}
		heapBitsSetType(uintptr(x), size, dataSize, typ)
        // 数组
		if dataSize > typ.size {
			// Array allocation. If there are any
			// pointers, GC has to scan to the last
			// element.
			if typ.ptrdata != 0 {
				scanSize = dataSize - typ.size + typ.ptrdata
			}
		} else {
			scanSize = typ.ptrdata
		}
		c.local_scan += scanSize
	}

	// Ensure that the stores above that initialize x to
	// type-safe memory and set the heap bits occur before
	// the caller can make x observable to the garbage
	// collector. Otherwise, on weakly ordered machines,
	// the garbage collector could follow a pointer to x,
	// but see uninitialized memory or stale heap bits.
	publicationBarrier()

	// Allocate black during GC.
	// All slots hold nil so no scanning is needed.
	// This may be racing with GC so do it atomically if there can be
	// a race marking the bit.
	if gcphase != _GCoff {
		gcmarknewobject(span, uintptr(x), size, scanSize)
	}

	if raceenabled {
		racemalloc(x, size)
	}

	if msanenabled {
		msanmalloc(x, size)
	}

	mp.mallocing = 0
	releasem(mp)

	if debug.allocfreetrace != 0 {
		tracealloc(x, size, typ)
	}

	if rate := MemProfileRate; rate > 0 {
		if rate != 1 && size < c.next_sample {
			c.next_sample -= size
		} else {
			mp := acquirem()
			profilealloc(mp, x, size)
			releasem(mp)
		}
	}

	if assistG != nil {
		// Account for internal fragmentation in the assist
		// debt now that we know it.
		assistG.gcAssistBytes -= int64(size - dataSize)
	}

	if shouldhelpgc {
		if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
			gcStart(t)
		}
	}

	return x
}


上述代码使用 runtime.gomcache 获取线程缓存并判断申请内存的类型是否为指针。我们从这个代码片段可以看出 runtime.mallocgc 会根据对象的大小执行不同的分配逻辑，在前面的章节也曾经介绍过运行时根据对象大小将它们分成微对象、小对象和大对象，这里会根据大小选择不同的分配逻辑：

微对象 (0, 16B) — 先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存；
小对象 [16B, 32KB] — 依次尝试使用线程缓存、中心缓存和堆分配内存；
大对象 (32KB, +∞) — 直接在堆上分配内存；
```





Go 语言运行时将**小于 16 字节的对象划分为微对象**，**它会使用线程缓存上的微分配器提高微对象分配的性能**，**我们主要使用它来分配较小的字符串以及逃逸的临时变量**。微**分配器可以将多个较小的内存分配请求合入同一个内存块中**，**只有当内存块中的所有对象都需要被回收时，整片内存才可能被回收。**

微分配器管理的对象不可以是指针类型，管理多个对象的内存块大小 `maxTinySize` 是可以调整的，在默认情况下，内存块的大小为 16 字节。`maxTinySize` 的值越大，组合多个对象的可能性就越高，内存浪费也就越严重；`maxTinySize` 越小，内存浪费就会越少，不过无论如何调整，8 的倍数都是一个很好的选择。

<img src="D:\gocode\read-code\images\2020-02-29-15829868066543-tiny-allocator.png" style="zoom:50%;" />

如上图所示，微分配器已经在 16 字节的内存块中分配了 12 字节的对象，如果下一个待分配的对象小于 4 字节，它会直接使用上述内存块的剩余部分，减少内存碎片，**不过该内存块只有所有对象都被标记为垃圾时才会回收**。



线程缓存 [`runtime.mcache`](https://draveness.me/golang/tree/runtime.mcache) 中的 `tiny` 字段指向了 `maxTinySize` 大小的块，如果当前块中还包含大小合适的空闲内存，运行时会通过基地址和偏移量获取并返回这块内存：



当**内存块中不包含空闲的内存时**，下面的这段代码会先从线程缓存找到跨度类对应的内存管理单元 [`runtime.mspan`](https://draveness.me/golang/tree/runtime.mspan)，调用 [`runtime.nextFreeFast`](https://draveness.me/golang/tree/runtime.nextFreeFast) 获取空闲的内存；当不存在空闲内存时，我们会调用 [`runtime.mcache.nextFree`](https://draveness.me/golang/tree/runtime.mcache.nextFree) 从中心缓存或者页堆中获取可分配的内存块：

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	...
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			...
			span := c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, _, _ = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			if size < c.tinyoffset || c.tiny == 0 {
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		}
		...
	}
	...
	return x
}
```





### 小对象

小对象是指大小为 16 字节到 32,768 字节的对象以及所有小于 16 字节的指针类型的对象，小对象的分配可以被分成以下的三个步骤：



1. 确定分配对象的大小以及跨度类 [`runtime.spanClass`](https://draveness.me/golang/tree/runtime.spanClass)；
2. 从线程缓存、中心缓存或者堆中获取内存管理单元并从内存管理单元找到空闲的内存空间；
3. 调用 [`runtime.memclrNoHeapPointers`](https://draveness.me/golang/tree/runtime.memclrNoHeapPointers) 清空空闲内存中的所有数据；
4. 确定待分配的对象大小以及跨度类需要使用预先计算好的 `size_to_class8`、`size_to_class128` 以及 `class_to_size` 字典，这些字典能够帮助我们快速获取对应的值并构建 [`runtime.spanClass`](https://draveness.me/golang/tree/runtime.spanClass)：



```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	...
	if size <= maxSmallSize {
		...
		} else {
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
			} else {
				sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			span := c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, _ = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
		...
	}
	...
	return x
}

这两个方法会帮助我们获取空闲的内存空间。runtime.nextFreeFast 会利用内存管理单元中的 allocCache 字段，快速找到该字段为 1 的位数，我们在上面介绍过 1 表示该位对应的内存空间是空闲的：
func nextFreeFast(s *mspan) gclinkptr {
	theBit := sys.Ctz64(s.allocCache)
	if theBit < 64 {
		result := s.freeindex + uintptr(theBit)
		if result < s.nelems {
			freeidx := result + 1
			if freeidx%64 == 0 && freeidx != s.nelems {
				return 0
			}
			s.allocCache >>= uint(theBit + 1)
			s.freeindex = freeidx
			s.allocCount++
			return gclinkptr(result*s.elemsize + s.base())
		}
	}
	return 0
}

找到了空闲的对象后，我们就可以更新内存管理单元的 allocCache、freeindex 等字段并返回该片内存；如果我们没有找到空闲的内存，运行时会通过 runtime.mcache.nextFree 找到新的内存管理单元：

func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
	s = c.alloc[spc]
	freeIndex := s.nextFreeIndex()
	if freeIndex == s.nelems {
		c.refill(spc)
		s = c.alloc[spc]
		freeIndex = s.nextFreeIndex()
	}

	v = gclinkptr(freeIndex*s.elemsize + s.base())
	s.allocCount++
	return
}
```





### 大对象

运行时对于大于 32KB 的大对象会单独处理，我们不会从线程缓存或者中心缓存中获取内存管理单元，而是直接调用 [`runtime.mcache.allocLarge`](https://draveness.me/golang/tree/runtime.mcache.allocLarge) 分配大片内存：

```go

runtime.mcache.allocLarge 会计算分配该对象所需要的页数，它按照 8KB 的倍数在堆上申请内存：

func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	...
	if size <= maxSmallSize {
		...
	} else {
		var s *mspan
		span = c.allocLarge(size, needzero, noscan)
		span.freeindex = 1
		span.allocCount = 1
		x = unsafe.Pointer(span.base())
		size = span.elemsize
	}

	publicationBarrier()
	mp.mallocing = 0
	releasem(mp)

	return x
}
```