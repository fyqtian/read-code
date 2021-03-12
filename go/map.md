### map

https://louyuting.blog.csdn.net/article/details/99699350

https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap

在通常情况下，哈希函数输入的范围一定会远远大于输出的范围，所以在使用哈希表时一定会遇到冲突，哪怕我们使用了完美的哈希函数，当输入的键足够多也会产生冲突。然而多数的哈希函数都是不够完美的，所以仍然存在发生哈希碰撞的可能，这时就需要一些方法来解决哈希碰撞的问题，常见方法的就是**开放寻址法和拉链法**。



<img src="..\images\image-20210203160709059.png" alt="image-20210203160709059" style="zoom:80%;" />

在一个性能比较好的哈希表中，每一个桶中都应该有 0~1 个元素，有时会有 2~3 个，很少会超过这个数量。**计算哈希、定位桶和遍历链表**三个过程是哈希表读写操作的主要开销，使用拉链法实现的哈希也有装载因子这一概念：**装载因子 = 元素数量 / 桶数量**

与开放地址法一样，**拉链法的装载因子越大，哈希的读写性能就越差**。在一般情况下使用拉链法的哈希表装载因子都不会超过 1，当哈希表的装载因子较大时会触发哈希的扩容，创建更多的桶来存储哈希中的元素，保证性能不会出现严重的下降。如果有 1000 个桶的哈希表存储了 10000 个键值对，它的性能是保存 1000 个键值对的 1/10，但是仍然比在链表中直接读写好 1000 倍。



<img src="..\images\image-20210203162439013.png" alt="image-20210203162439013" style="zoom:67%;" />



<img src="..\images\20190907092107290.png" alt="20190907092107290" style="zoom: 50%;" />



<img src="..\images\20190907100725824.png" alt="20190907100725824" style="zoom:67%;" />



当 map 的 **key 和 value 都不是指针，并且 size 都小于 128 字节的情况下**，会把 bmap 标记为不含指针，这样可以避免 gc 时扫描整个 hmap。但是，我们看 bmap 其实有一个 overflow 的字段，是指针类型的，破坏了 bmap 不含指针的设想，这时会把 overflow 移动到 extra 字段来。



如果key是float类型的会造成？

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
	nevacuate  uintptr        // //疏散进度计数器（小于此值的桶已疏散）

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


创建流程
1. 先通过容量计算需要多少个桶B一个桶8个元素，
	计算B 元素数量>8 && 元素数量/桶数量 > 6.5
	如果B>=4就分配（b-4）个溢出桶，溢出桶和正常桶是连续的地址空间
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

访问流程
1.如果有写标记panic，key hash定位到桶
2.如果存在老桶判断是同等扩容还是增量扩容，如果是增量扩容 找到老桶位置 通过top[0]判断是否迁移完毕 如果未迁移在老桶里查找
3.通过比对key的高8位和桶的top[8]，判断是否存在key，如果存在取桶内第几个位置的key再做比较如果相同返回，失败下一轮
4.如果当前桶未找到，如果有溢出桶继续直到没有溢出桶。

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
    //计算高8位 一个byte
	top := tophash(hash)
bucketloop:
    //从当前桶找到溢出桶直到Nil
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
            //找到相同的top 比对key
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
            //比对key失败 下一轮
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


00018 (+5) CALL runtime.
mapassign_fast64(SB)
00020 (5) MOVQ 24(SP), DI               ;; DI = &value
00026 (5) LEAQ go.string."88"(SB), AX   ;; AX = &"88"
00027 (5) MOVQ AX, (DI)                 ;; *DI = AX


1.判断flags&hashWriting != 0
2.计算hash，如果桶为nil，创建桶，hash低B位定位到桶，如果在迁移，迁移两个桶
3.通过hash的高8位在桶里查找，如果当前的top[index] == emptyRest，写入位置
	否则比对key如果相等，更新value，结束
	直到溢出桶为空
4. 如果不在扩容期间判断是否需要同等扩容，还是翻倍扩容，如果扩容重新走一遍流程
5. 如果没有找到写入位置 申请溢出桶
6. 写入h.count++ 移除写flag
7. 返回value写入地址，汇编处理实际写入




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
    //todo
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
    //todo	
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



// tooManyOverflowBuckets reports whether noverflow buckets is too many for a map with 1<<B buckets.
// Note that most of these overflow buckets must be in sparse use;
// if use was dense, then we'd have already triggered regular map growth.
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// If the threshold is too low, we do extraneous work.
	// If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
	// "too many" means (approximately) as many overflow buckets as regular buckets.
	// See incrnoverflow for more details.
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}


1.如果在迁移状态，迁移两个桶
2.定位到桶
3.如果top[index] == emptyRest 说明后面已经没有key 跳出搜索循环，否则比对key如果相等移除key，value
	当前top[index]=emptyOne
	如果当前的index是桶最后一个元素，还存在溢出桶并且溢出桶的top[0] != emptyRest 结束
	如果不是最后个元素判断下个元素是不是emptyRest，如果不是结束
		当前top[index]=emptyRest,向前遍历如果前一个top[index] != emptyOne跳出,否则不停的向上一个桶遍历
4.恢复flag 移除写标记位





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
                // 当前不是最后一个桶 并且下个桶也不是
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







```go
//扩容
overflow 的 bucket 数量过多，这有两种情况：
载因子超过阈值，源码里定义的阈值是 6.5。
当 B 大于15时，也就是 bucket 总数大于 2^15 时，如果overflow的bucket数量大于2^15，就触发扩容。
当B小于15时，如果overflow的bucket数量大于2^B 也会触发扩容。


第 1 点：我们知道，每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。

第 2 点：是对第 1 点的补充。就是说在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）。

不难想像造成这种情况的原因：不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，但就是不会触犯第 1 点的规定，你能拿我怎么办？**overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人，**因此出台第 2 点规定。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。

// 可能有迭代器使用 buckets
iterator     = 1
// 可能有迭代器使用 oldbuckets
oldIterator  = 2
// 有协程正在并发的向 map 中写入 key
hashWriting  = 4
// 等量扩容（对应条件 2）
sameSizeGrow = 8


func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
    // 如果是负载因子太大就扩容 否则就是同等扩容
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
    //复制旧桶
	oldbuckets := h.buckets
    //创建新桶和溢出
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
	// 清空标记?
	flags := h.flags &^ (iterator | oldIterator)
    //先把 h.flags 中 iterator 和 oldIterator 对应位清 0，然后如果发现 iterator 位为 1，那就把它转接到 oldIterator 位，使得 oldIterator 标志位变成 1。潜台词就是：buckets 现在挂到了 oldBuckets 名下了，对应的标志位也转接过去。

	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
    // 更新字段
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0    //搬迁进度
	h.noverflow = 0
	//如果溢出桶不为空
	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
        //溢出桶挂道旧的溢出桶字段上
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
        //挂上新的溢出桶
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}


桶的扩容 hashgrow 只在写入的时候触发
桶迁移 growWork map删除和map写入时触发

func growWork(t *maptype, h *hmap, bucket uintptr) {
  // 确认搬迁老的 bucket 对应正在使用的 bucket
    这行代码，如源码注释里说的，是为了确认搬迁的 bucket 是我们正在使用的 bucket。oldbucketmask() 函数返回扩容前的 map 的 bucketmask。
	evacuate(t, h, bucket&h.oldbucketmask())

	// 再搬迁一个 bucket，以加快搬迁进程
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}

}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
    //老桶地址
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
   // 分配了多少桶 用来做偏移量
    // 当前桶的最后一个下标
	newbit := h.noldbuckets()
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		// X和Y分别代表，如果是2倍扩容时，对应的前半部分和后半部分
        var xy [2]evacDst
		x := &xy[0]
        //旧桶地址
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
        //旧桶key开始的位置  
		x.k = add(unsafe.Pointer(x.b), dataOffset)
        // 旧桶value开始的位置
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))
		// 如果是翻倍扩容
		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
          	// 如果不是等 size 扩容，前后 bucket 序号有变
			// 使用 y 来进行搬迁
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}
		//遍历桶直到溢出桶为nil
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
				top := b.tophash[i]
                // 如果top<= emptyOne 标记为evacuatedEmpty迁移完成
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
                // 正常不会出现这种情况
				// 未被搬迁的 cell 只可能是 empty 或是
				// 正常的 top hash（大于 minTopHash）
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
                //key是指针
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
                //如果不是等量扩容 要计算hash低的B位是否要去新桶
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/elem to bucket x or bucket y).
                    
					hash := t.hasher(k2, uintptr(h.hash0))
                    	// // 如果有协程正在遍历 map 且出现 相同的 key 值，算出来的 hash 值不同
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
                        
                        float类型
                        
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
                        // 使用新桶
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}
				// 找到实际要去的桶
				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination
				//如果已经要新桶的最后一个位置 申请一个溢出桶
				if dst.i == bucketCnt {
                    //挂一个溢出桶
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
                // 
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
                //复制key
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
                // 复制value
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
				dst.i++
				// These updates might push these pointers past the end of the
				// key or elem arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
		// Unlink the overflow buckets & clear key/elem to help GC.
        // 如果没有协程在使用老的 buckets，就把老 buckets 清除掉，帮助gc
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}
	// 迁移记录+1
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}



func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
    //迁移进度+1
	h.nevacuate++
	// Experiments suggest that 1024 is overkill by at least an order of magnitude.
	// Put it in there as a safeguard anyway, to ensure O(1) behavior.
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
    // 如果迁移进度已经到最后一个桶
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
		h.oldbuckets = nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}
```





### 为什么遍历 map 是无序的？

map 在扩容后，**会发生 key 的搬迁**，原来落在同一个 bucket 中的 key，搬迁后，有些 key 就要远走高飞了（bucket 序号加上了 2^B）。而遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。搬迁后，key 的位置发生了重大的变化，有些 key 飞上高枝，有些 key 则原地不动。这样，遍历 map 的结果就不可能按原来的顺序了。

当然，Go 做得更绝，当我们在遍历 map 时，**并不是固定地从 0 号 bucket 开始遍历**，**每次都是从一个随机值序号的 bucket 开始遍历**，**并且是从这个 bucket 的一个随机序号的 cell 开始遍历**。这样，即使你是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对了。








```go
// mapiterinit initializes the hiter struct used for ranging over maps.
// The hiter struct pointed to by 'it' is allocated on the stack
// by the compilers order pass or on the heap by reflect_mapiterinit.
// Both need to have zeroed hiter since the struct contains pointers.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
   if h == nil || h.count == 0 {
      return
   }

   if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
      throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
   }
   it.t = t
   it.h = h

   // grab snapshot of bucket state
   it.B = h.B
   it.buckets = h.buckets
   if t.bucket.ptrdata == 0 {
      // Allocate the current slice and remember pointers to both current and old.
      // This preserves all relevant overflow buckets alive even if
      // the table grows and/or overflow buckets are added to the table
      // while we are iterating.
      h.createOverflow()
      it.overflow = h.extra.overflow
      it.oldoverflow = h.extra.oldoverflow
   }

   // decide where to start
   r := uintptr(fastrand())
   if h.B > 31-bucketCntBits {
      r += uintptr(fastrand()) << 31
   }
    //随机一个bucket
   it.startBucket = r & bucketMask(h.B)
   it.offset = uint8(r >> h.B & (bucketCnt - 1))

   // iterator state
   it.bucket = it.startBucket

   // Remember we have an iterator.
   // Can run concurrently with another mapiterinit().
   if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
      atomic.Or8(&h.flags, iterator|oldIterator)
   }

   mapiternext(it)
}


func mapiternext(it *hiter) {
	h := it.h
	
	if h.flags&hashWriting != 0 {
		throw("concurrent map iteration and map write")
	}
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.elem = nil
			return
		}
        //在同等扩容
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
            //未迁移完 
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
    //遍历桶内元素
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// TODO: emptyRest is hard to use here, as we start iterating
			// in the middle of a bucket. It's feasible, just tricky.
			continue
		}
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey() {
			k = *((*unsafe.Pointer)(k))
		}
		e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
		if checkBucket != noCheck && !h.sameSizeGrow() {
			// Special case: iterator was started during a grow to a larger size
			// and the grow is not done yet. We're working on a bucket whose
			// oldbucket has not been evacuated yet. Or at least, it wasn't
			// evacuated when we started the bucket. So we're iterating
			// through the oldbucket, skipping any keys that will go
			// to the other new bucket (each oldbucket expands to two
			// buckets during a grow).
			if t.reflexivekey() || t.key.equal(k, k) {
				// If the item in the oldbucket is not destined for
				// the current new bucket in the iteration, skip it.
				hash := t.hasher(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
				// Hash isn't repeatable if k != k (NaNs).  We need a
				// repeatable and randomish choice of which direction
				// to send NaNs during evacuation. We'll use the low
				// bit of tophash to decide which way NaNs go.
				// NOTE: this case is why we need two evacuate tophash
				// values, evacuatedX and evacuatedY, that differ in
				// their low bit.
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || t.key.equal(k, k)) {
			// This is the golden data, we can return it.
			// OR
			// key!=key, so the entry can't be deleted or updated, so we can just return it.
			// That's lucky for us because when key!=key we can't look it up successfully.
			it.key = k
			if t.indirectelem() {
				e = *((*unsafe.Pointer)(e))
			}
			it.elem = e
		} else {
			// The hash table has grown since the iterator was started.
			// The golden data for this key is now somewhere else.
			// Check the current hash table for the data.
			// This code handles the case where the key
			// has been deleted, updated, or deleted and reinserted.
			// NOTE: we need to regrab the key as it has potentially been
			// updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
			rk, re := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.elem = re
		}
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```