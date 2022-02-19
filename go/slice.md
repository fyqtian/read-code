### Slice

https://juejin.cn/post/6844903812331732999

```go
https://juejin.cn/post/6844903812331732999
https://louyuting.blog.csdn.net/article/details/99608199
// SliceHeader在运行时表示一个slice
// 它不能安全或便携的使用，因为将来可能会变更
// 而且，Data字段并不足够保证这个数组不会被收集，因此程序必须保持单独的正确的指针指向底层数据
type SliceHeader struct {
   Data uintptr   // 指向数组的指针;
   Len  int		  // 当前切片的长度
   Cap  int       // 当前切片的容量，即 Data 数组的大小
}

// 通过数组字面量创建slice
func main() {
	arr := [3]int{100, 200, 300}
	slice := arr[0:1]
	ca(slice)
}
func ca([]int) {}
	0x0039 00057 (slice.go:4)	MOVQ	$100, "".arr+24(SP)
	0x0042 00066 (slice.go:4)	MOVQ	$200, "".arr+32(SP)
	0x004b 00075 (slice.go:4)	MOVQ	$300, "".arr+40(SP)
	0x0054 00084 (slice.go:5)	LEAQ	"".arr+24(SP), AX  //数组地址
	0x0059 00089 (slice.go:5)	TESTB	AL, (AX)
	0x005b 00091 (slice.go:5)	JMP	93
	0x005d 00093 (slice.go:5)	JMP	95
	0x005f 00095 (slice.go:5)	MOVQ	AX, "".slice+48(SP) //Slice.Data
	0x0064 00100 (slice.go:5)	MOVQ	$1, "".slice+56(SP) //Slice.Len
	0x006d 00109 (slice.go:5)	MOVQ	$3, "".slice+64(SP) //Slice.Cap
	0x0076 00118 (slice.go:6)	MOVQ	AX, (SP)    		//ca参数
	0x007a 00122 (slice.go:6)	MOVQ	$1, 8(SP)			//ca参数
	0x0083 00131 (slice.go:6)	MOVQ	$3, 16(SP)          //ca参数
	0x008c 00140 (slice.go:6)	PCDATA	$1, $0
	0x008c 00140 (slice.go:6)	CALL	"".ca(SB)


// 通过make创建
func main() {
    //容量太小了 还是会在栈上直接分配
	m := make([]int, 4096*2)
	m[0] = 999
}

	0x0024 00036 (slice2.go:4)	LEAQ	type.int(SB), AX      //AX = &_type(int)
	0x002b 00043 (slice2.go:4)	MOVQ	AX, (SP)		      //makeslice arg0
	0x002f 00047 (slice2.go:4)	MOVQ	$8192, 8(SP)		  //makeslice arg1	
	0x0038 00056 (slice2.go:4)	MOVQ	$8192, 16(SP)         //makeslice arg2
	0x0041 00065 (slice2.go:4)	PCDATA	$1, $0
	0x0041 00065 (slice2.go:4)	CALL	runtime.makeslice(SB) 
	0x0046 00070 (slice2.go:4)	MOVQ	24(SP), AX            //AX = makeslice()
	0x004b 00075 (slice2.go:4)	MOVQ	AX, "".m+32(SP)       //m.Data = AX 
	0x0050 00080 (slice2.go:4)	MOVQ	$8192, "".m+40(SP)    //m.Len = 8192
	0x0059 00089 (slice2.go:4)	MOVQ	$8192, "".m+48(SP)    //m.Cap = 8192
	0x0062 00098 (slice2.go:5)	JMP	100
	0x0064 00100 (slice2.go:5)	MOVQ	$999, (AX)            //m[0] = 999  
	0x006b 00107 (slice2.go:6)	MOVQ	56(SP), BP
	0x0070 00112 (slice2.go:6)	ADDQ	$64, SP
	0x0074 00116 (slice2.go:6)	RET



func makeslice(et *_type, len, cap int) unsafe.Pointer {
    //计算内存和溢出
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
    // 溢出 || 内存>限制 || len <0 || len > cap
	if overflow || mem > maxAlloc || len < 0 || len > cap {
        // 注意当make([]T, bignumber)产生一个'len out of range'错误而不是'cap out of range'的错误
        //	'cap out of range'也是准确的，但是cap是隐式提供的，用len更准确
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}
	// 申请内存
	return mallocgc(mem, et, true)
}


// 扩容
func main() {
	m := make([]int, 8192)
	m = append(m, 100)
}
	0x0028 00040 (slice3.go:4)	LEAQ	type.int(SB), AX       // AX = &_type
	0x002f 00047 (slice3.go:4)	MOVQ	AX, (SP)               //makeslice arg0
	0x0033 00051 (slice3.go:4)	MOVQ	$8192, 8(SP)		   //makeslice arg1
	0x003c 00060 (slice3.go:4)	MOVQ	$8192, 16(SP)          //makeslice arg2
	0x0045 00069 (slice3.go:4)	PCDATA	$1, $0
	0x0045 00069 (slice3.go:4)	CALL	runtime.makeslice(SB)
	0x004a 00074 (slice3.go:4)	MOVQ	24(SP), AX             // AX = makeslice()
	0x004f 00079 (slice3.go:4)	MOVQ	AX, "".m+64(SP)        // m.Data = AX
	0x0054 00084 (slice3.go:4)	MOVQ	$8192, "".m+72(SP)     // m.Len = 8192
	0x005d 00093 (slice3.go:4)	MOVQ	$8192, "".m+80(SP)     // m.Cap = 8192
	0x0066 00102 (slice3.go:5)	JMP	104
	0x0068 00104 (slice3.go:5)	LEAQ	type.int(SB), CX       // CX = &_type(int)
	0x006f 00111 (slice3.go:5)	MOVQ	CX, (SP)			   // growslice arg0
	0x0073 00115 (slice3.go:5)	MOVQ	AX, 8(SP)              // growslcie arg1=slice.Data
	0x0078 00120 (slice3.go:5)	MOVQ	$8192, 16(SP)          // growslcie arg1=slice.Len  
	0x0081 00129 (slice3.go:5)	MOVQ	$8192, 24(SP)          // growslcie arg1=slice.Cap  
	0x008a 00138 (slice3.go:5)	MOVQ	$8193, 32(SP)          // growslcie arg2
	0x0093 00147 (slice3.go:5)	CALL	runtime.growslice(SB)
	0x0098 00152 (slice3.go:5)	MOVQ	40(SP), AX             //glowsilce rs.Data
	0x009d 00157 (slice3.go:5)	MOVQ	48(SP), CX             //glowsilce rs.Len
	0x00a2 00162 (slice3.go:5)	MOVQ	56(SP), DX             //glowsilce rs.Cap
	0x00a7 00167 (slice3.go:5)	INCQ	CX                     // rs.Len+1
	0x00aa 00170 (slice3.go:5)	JMP	172
	0x00ac 00172 (slice3.go:5)	MOVQ	$100, 65536(AX)        //65536 = 8192*8得位置写入100
	0x00b7 00183 (slice3.go:5)	MOVQ	AX, "".m+64(SP)        //替换旧得slice.Data
	0x00bc 00188 (slice3.go:5)	MOVQ	CX, "".m+72(SP)		   //替换旧得slice.Len
	0x00c1 00193 (slice3.go:5)	MOVQ	DX, "".m+80(SP)        //替换旧得slice.Cap
	0x00c6 00198 (slice3.go:6)	MOVQ	88(SP), BP
	0x00cb 00203 (slice3.go:6)	ADDQ	$96, SP


// growslice处理slice的append期间得扩容
// 通过传递slice的元素类型，旧的slice，期望的最小容量
// 它返回一个新的slice至少有期望的容量和旧的数据
// 新的slice的长度被设置为旧的slice的长度，并不是新请求的容量
// 这是为了方便代码生成？，旧slice的容量被立即使用来计算扩容后的长度
func growslice(et *_type, old slice, cap int) slice {
	// 如果新容量 < 旧slice的容量
	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}
	// 如果类型的长度是空
	if et.size == 0 {
        // append不应创建一个nil指针长度不为0的slice
        // 我们假设append不需要保存旧的数据
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}
	//旧slice的CAP
	newcap := old.cap
    //容量翻倍
	doublecap := newcap + newcap
    //如果申请容量 > 两倍的旧容量
	if cap > doublecap {
        // 新容量 = 申请容量
		newcap = cap
	} else {
        //如果旧容量 < 1024 新容量 = 旧容量 * 2
		if old.len < 1024 {
			newcap = doublecap
		} else {
            // 如果旧容量>0 && 旧容量< 申请容量 && 旧容量 > 1024
			for 0 < newcap && newcap < cap {
                // 新容量 = 旧容量 * 1.25 循环，直到新容量 > 申请容量
				newcap += newcap / 4
			}
			// 新容量溢出
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
    // 对不同类型进行计算需要扩容的空间 会有向上取整的操作
    // 对于et.size是1 我们不需要做任何除法或乘法
    // 对于sys.PtrSize,编译器会优化除法乘法为一个位移
    // 对于2的幂,使用可变移位
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}
    // 如果溢出
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}
	
	var p unsafe.Pointer
    //如果元素不是指针类型
	if et.ptrdata == 0 {
        // 申请内存
		p = mallocgc(capmem, nil, false)
        // 先将 P 地址加上新的容量得到新切片容量的地址，然后将新切片容量地址后面的 capmem-newlenmem 个 bytes 这块内存初始化。为之后继续 append() 操作腾出空间。
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
        // 注意 不能使用原始的内存（避免空内存？），因为GC可以扫描到未初始化的内存
		p = mallocgc(capmem, et, true)
        // 不知道是干啥的
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
		}
	}
    // 旧切片的数据 复制到新切片上
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}



内存拷贝
如果slice 容量太小会直接优化


func slicecopy(toPtr unsafe.Pointer, toLen int, fmPtr unsafe.Pointer, fmLen int, width uintptr) int {
	
    if fmLen == 0 || toLen == 0 {
		return 0
	}
	// 选两个切片长度较小的
	n := fmLen
	if toLen < n {
		n = toLen
	}
	// 如果入参 width = 0，也不需要拷贝了，返回较短的切片的长度
	if width == 0 {
		return n
	}
	// 总共拷贝的大小
	size := uintptr(n) * width
	if size == 1 { // common case worth about 2x to do here
		// TODO: is this still worth it with new memmove impl?
		*(*byte)(toPtr) = *(*byte)(fmPtr) // known to be a byte pointer
	} else {
		memmove(toPtr, fmPtr, size)
	}
	return n
}

```

