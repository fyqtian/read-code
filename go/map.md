### map

在通常情况下，哈希函数输入的范围一定会远远大于输出的范围，所以在使用哈希表时一定会遇到冲突，哪怕我们使用了完美的哈希函数，当输入的键足够多也会产生冲突。然而多数的哈希函数都是不够完美的，所以仍然存在发生哈希碰撞的可能，这时就需要一些方法来解决哈希碰撞的问题，常见方法的就是**开放寻址法和拉链法**。



<img src="..\images\image-20210203160709059.png" alt="image-20210203160709059" style="zoom:80%;" />

在一个性能比较好的哈希表中，每一个桶中都应该有 0~1 个元素，有时会有 2~3 个，很少会超过这个数量。**计算哈希、定位桶和遍历链表**三个过程是哈希表读写操作的主要开销，使用拉链法实现的哈希也有装载因子这一概念：**装载因子 = 元素数量 / 桶数量**

与开放地址法一样，**拉链法的装载因子越大，哈希的读写性能就越差**。在一般情况下使用拉链法的哈希表装载因子都不会超过 1，当哈希表的装载因子较大时会触发哈希的扩容，创建更多的桶来存储哈希中的元素，保证性能不会出现严重的下降。如果有 1000 个桶的哈希表存储了 10000 个键值对，它的性能是保存 1000 个键值对的 1/10，但是仍然比在链表中直接读写好 1000 倍。



<img src="..\images\image-20210203162439013.png" alt="image-20210203162439013" style="zoom:67%;" />



<img src="..\images\20190907092107290.png" alt="20190907092107290" style="zoom: 50%;" />



<img src="..\images\20190907100725824.png" alt="20190907100725824" style="zoom:67%;" />

```go
//容量太小 编译器会优化
func main() {
   m := make(map[string]string，8192)
   m["f"] = "first"
   m["s"] = "second"
}
	0x0028 00040 (map1.go:4)	XORPS	X0, X0
	0x002b 00043 (map1.go:4)	MOVUPS	X0, ""..autotmp_1+64(SP)
	0x0030 00048 (map1.go:4)	MOVUPS	X0, ""..autotmp_1+80(SP)
	0x0035 00053 (map1.go:4)	MOVUPS	X0, ""..autotmp_1+96(SP)
	0x003a 00058 (map1.go:4)	LEAQ	type.map[string]string(SB), AX   // AX = _type(map[string][string])
	0x0041 00065 (map1.go:4)	MOVQ	AX, (SP)			 //makemap arg0			 					
	0x0045 00069 (map1.go:4)	MOVQ	$8192, 8(SP)         //makesmap arg1 
	0x004e 00078 (map1.go:4)	LEAQ	""..autotmp_1+64(SP), AX // *hmap
	0x0053 00083 (map1.go:4)	MOVQ	AX, 16(SP)           //makemap arg2
	0x0058 00088 (map1.go:4)	PCDATA	$1, $0
	0x0058 00088 (map1.go:4)	CALL	runtime.makemap(SB)
	0x005d 00093 (map1.go:4)	MOVQ	24(SP), AX           // AX = makemap()
	0x0062 00098 (map1.go:4)	MOVQ	AX, "".m+40(SP)      // m = AX
	0x0067 00103 (map1.go:5)	LEAQ	type.map[string]string(SB), CX    
	0x006e 00110 (map1.go:5)	MOVQ	CX, (SP)             // mapassign_faststr arg0
	0x0072 00114 (map1.go:5)	MOVQ	AX, 8(SP)           // mapassign_faststr arg1 makemap return address
	0x0077 00119 (map1.go:5)	LEAQ	go.string."f"(SB), AX // mapassign_faststr arg2 s.Data
	0x007e 00126 (map1.go:5)	MOVQ	AX, 16(SP)          // mapassign_faststr arg2 s.Len
	0x0083 00131 (map1.go:5)	MOVQ	$1, 24(SP)           // mapassign_faststr arg2 s.Cap
	0x008c 00140 (map1.go:5)	PCDATA	$1, $1
	0x008c 00140 (map1.go:5)	CALL	runtime.mapassign_faststr(SB) 
	0x0091 00145 (map1.go:5)	MOVQ	32(SP), DI           // rs = mapassign_faststr() value写入得位置
	0x0096 00150 (map1.go:5)	MOVQ	DI, ""..autotmp_2+56(SP)
	0x009b 00155 (map1.go:5)	TESTB	AL, (DI)
	0x009d 00157 (map1.go:5)	MOVQ	$5, 8(DI)            //
	0x00a5 00165 (map1.go:5)	PCDATA	$0, $-2
	0x00a5 00165 (map1.go:5)	CMPL	runtime.writeBarrier(SB), $0
	0x00ac 00172 (map1.go:5)	JEQ	176
	0x00ae 00174 (map1.go:5)	JMP	303
	0x00b0 00176 (map1.go:5)	LEAQ	go.string."first"(SB), AX
	0x00b7 00183 (map1.go:5)	MOVQ	AX, (DI)               // insert map
	0x00ba 00186 (map1.go:5)	JMP	188
	.....
	0x0120 00288 (map1.go:7)	RET


// A header for a Go map.
type hmap struct {
	count     int // # 元素个数 live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // 桶数量 B的2次方
	noverflow uint16 // 桶溢出数量
	hash0     uint32 // 哈希种子
	buckets    unsafe.Pointer // array of 2^B Buckets. 桶地址
	oldbuckets unsafe.Pointer // 旧桶地址 当前桶/2
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}


// 编译期间类型 cmd/compile/internal/gc.bmap 
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}

// runtime bmap 桶
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}

// mapextra 保存字段并不是所有map都有
type mapextra struct {
    // 如果key和elem都不包含指针并且是内联得，我们标记bucket不包含指针
    // 避免扫描map， 然后bmap.overflow是一个指针，为了保持overflow存活，我们保存指向一处同得指针在hmap.extra.overflow
    // 和 hmap.extra.oldoverflow.overflow和oldvoerflow仅在key和elem不包含指针得时候
    // overflow包含溢出得桶hmap.buckets.
    // oldoverflow包含溢出得hmap.oldbuckes
	overflow    *[]*bmap
	oldoverflow *[]*bmap
    // nextoverflow持有空闲得溢出桶指针
	nextOverflow *bmap
}


type maptype struct {
	typ    _type
	key    *_type
	elem   *_type
	bucket *_type // internal type representing a hash bucket
	// function for hashing keys (ptr to key, seed) -> hash
	hasher     func(unsafe.Pointer, uintptr) uintptr
	keysize    uint8  // size of key slot
	elemsize   uint8  // size of elem slot
	bucketsize uint16 // size of bucket
	flags      uint32
}


// makemap实现GO创建make(map[k]v,hint)
// 如果编译器已经决定map第一个bucket可以创建在栈上，h或者bucket可能是非nil
// 如果 h != nil map可以被h直接创建
// 如果 h.bucket不为空 bucket指向的地址可以被用来做第一个bucket
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // 如果是string 一个string 24个字节 一个桶是8个string 272？
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}
    // 复用map
	if h == nil {
		h = new(hmap)
	}
    // hash种子
	h.hash0 = fastrand()
    // 找到可以存放下hint数量的B 比如8192个元素 需要2042个桶
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

    // 分配初始化hash table 
    // 如果B == 0 bucket字段将会在mapassign时候分配
    // 如果hint很多清空内存将消耗一段时间
	if h.B != 0 {
		var nextOverflow *bmap
        //桶地址
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		//溢出桶不为空
        if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

 // makeBucketArray初始化map得映射后得数组
 //  1<<b 是分配得最小桶数量
 //  dirtyalloc可能是nil或者先前得bucket
 // 通过makeBucketArray分配拥有相同得t和b参数分配
 // 如果dirtyalloc是nil一个新得后背数组将会被分配否则dirtyalloc将会被清空等待复用
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// 对于较小的B 不太可能溢出
	// 避免计算开销
	if b >= 4 {
        // 计算估计得溢出桶数量，需要插入的中位数
		nbuckets += bucketShift(b - 4)
        // 溢出的容量
		sz := t.bucket.size * nbuckets
        //向上取整
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}
	//申请内存
	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
        // dirtyalloc是先前生成的 可能不为空
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
        //内存重置为0
		if t.bucket.ptrdata != 0 {
            // 如果map含有指针
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}
	// 如果申请了 溢出桶
	if base != nbuckets {
        // 我们预先分配了一些溢出桶
        // 为了将追踪这些桶的开销降到最低
        // 我们使用约定预先分配的溢出桶指针是空，then there are more available by bumping the pointer.
        // 我们需要安全的非空指针对于最后的溢出桶
       
        //溢出桶的位置
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
        //最后个桶的位置
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
        //(*bmap)(buckets)是桶得首地址
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}


// m := map[k]v{}
// val := m[key]
// mapaccess1返回指向h[key]得指针，当key不在map时从不返回Nil, 相反会返回一个对应得空对象
// 注意 返回得Pointer可能保持在map得生命周期，因此不要保持太长
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 如果为空 返回空值
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
    //并发写 异常
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
    //key得hash
	hash := t.hasher(key, uintptr(h.hash0))
    //桶得1<<h.b - 1
	m := bucketMask(h.B)
    //计算桶位置 hash & (1<<h.b - 1)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    // 如果存在老桶
	if c := h.oldbuckets; c != nil {
        // 不是同容量扩容 当前得桶mask右移一位 相当于h.B-1（当前容量得一半为老桶）
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
        // key在老桶中得位置
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        //计算key是否还在老桶中
		if !evacuated(oldb) {
			b = oldb
		}
	}
    //计算高8位
	top := tophash(hash)
bucketloop:
    //从当前桶找到溢出桶知道nil
	for ; b != nil; b = b.overflow(t) {
        //bucketCnt = 8
		for i := uintptr(0); i < bucketCnt; i++ {
			//从0下标开始比对top
            if b.tophash[i] != top {
                // 如果是emptyRest 说明后面都为空 不再遍历
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
            //找到key
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            // 如果是指针
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
            //判断 key == k
			if t.key.equal(key, k) {
                //找到value
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				// 如果是指针
                if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}

// 判断这个 bucket 是否已经搬迁完毕，
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > emptyOne && h < minTopHash
}

下面的这几种状态就表征了 bucket 的情况：
emptyRest      = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
emptyOne       = 1 // this cell is empty
// 扩容相关
evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
// 扩容相关
evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
minTopHash     = 5 // minimum tophash for a normal filled cell.


00018 (+5) CALL runtime.mapassign_fast64(SB)
00020 (5) MOVQ 24(SP), DI               ;; DI = &value
00026 (5) LEAQ go.string."88"(SB), AX   ;; AX = &"88"
00027 (5) MOVQ AX, (DI)                 ;; *DI = AX

// 如果当前key不存在map中分配一个slot
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	// 并发写
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
    //计算hash
	hash := t.hasher(key, uintptr(h.hash0))
	// 设置map标志位 正在写
	h.flags ^= hashWriting
	// 初始化桶
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
    // 计算桶位置
	bucket := hash & bucketMask(h.B)
    // 如果老桶不为空 表示正在迁移
    // 每次迁移两个桶
	if h.growing() {
		growWork(t, h, bucket)
	}
    // key所在得桶地址
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
            //key得hash高8位比较
			if b.tophash[i] != top {
                //如果b.tophash <= 1 
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
                    // k写入位置
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					// elem写入位置
                    elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
                //如果tophash得位置是空得 跳出循环
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
            //如果tophash已经被设置 比对两个key
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			//如果是指针解引用
            if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
            //如果两个可以不相等 继续循环
			if !t.key.equal(key, k) {
				continue
			}
			// map已经存在得key 不清楚为什么要更新key？
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
            //计算写入elem得位置 流程结束
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
		}
        //计算溢出桶
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}
	// 当前得桶未找到可以存放key的位置，分配一个新的空间
	// 如果我们达到最大负载系数或者我们有太多的溢出桶，并且我们并不在一个扩容中，开始扩容
    //装载因子已经超过 6.5；
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
	// 如果还未找到存放的位置 创建溢出桶
	if inserti == nil {
		// all current buckets are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// store new key/elem at insert position
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
    // 移除标记位
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
// 获取溢出桶
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
	var ovf *bmap
	if h.extra != nil && h.extra.nextOverflow != nil {
        // 存在预先分配的溢出桶
		ovf = h.extra.nextOverflow
		if ovf.overflow(t) == nil {
			// 还未到达预先分配的最后一个桶  更新溢出桶指针
			h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
		} else {
            // 这是最后一个溢出桶 重置溢出指针
			ovf.setoverflow(t, nil)
            // 下次在使用溢出桶 需要重新分配
			h.extra.nextOverflow = nil
		}
	} else {
        // 申请一个溢出桶
		ovf = (*bmap)(newobject(t.bucket))
	}
    //溢出数量+1
	h.incrnoverflow()
    // 如果不含指针
	if t.bucket.ptrdata == 0 {
		h. createOverflow()
        // 插入到overflow
		*h.extra.overflow = append(*h.extra.overflow, ovf)
	}
    // 把溢出桶挂在当前桶后面
	b.setoverflow(t, ovf)
	return ovf
}



func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}

// 删除key
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	hash := t.hasher(key, uintptr(h.hash0))
	// 修改状态
	h.flags ^= hashWriting
	bucket := hash & bucketMask(h.B)
    //如果正在迁移 迁移两个桶
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
                //如果标记emptyRest 说明后面没有key 跳出
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
            // 是否为指针
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
            // 比较可以是否相等
			if !t.key.equal(key, k2) {
				continue
			}
			// 如果是指针类型的清空
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
            // 清理value
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}
            //tophash状态变更为空
			b.tophash[i] = emptyOne
            // 如果当前的桶以一堆emptyOne状态结束，变更状态为emptyRest
            // 如果下标是7 桶最后一个位置
			if i == bucketCnt-1 {
                // 如果还有溢出桶 并且溢出桶第一个并不是emptyRest
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
                // 当前不是最后一个桶 并且下个桶 也不是
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
                // 向前更新桶状态
				b.tophash[i] = emptyRest
				if i == 0 {
                    // 到达第一个桶
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// 向前找桶
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
            // 元素数量减1
			h.count--
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}

```