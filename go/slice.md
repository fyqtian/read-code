### Slice

```go
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
	0x00ac 00172 (slice3.go:5)	MOVQ	$100, 65536(AX)        //
	0x00b7 00183 (slice3.go:5)	MOVQ	AX, "".m+64(SP)        //替换旧得slice.Data
	0x00bc 00188 (slice3.go:5)	MOVQ	CX, "".m+72(SP)		   //替换旧得slice.Len
	0x00c1 00193 (slice3.go:5)	MOVQ	DX, "".m+80(SP)        //替换旧得slice.Cap
	0x00c6 00198 (slice3.go:6)	MOVQ	88(SP), BP
	0x00cb 00203 (slice3.go:6)	ADDQ	$96, SP


```

