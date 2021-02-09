### interface

//汇编
https://www.cnblogs.com/yjf512/p/6132868.html
https://blog.csdn.net/tencent_teg/article/details/108413692
https://zhuanlan.zhihu.com/p/56750445
https://blog.csdn.net/u010853261/article/details/101060546
https://www.codercto.com/a/103852.html
https://zhuanlan.zhihu.com/p/27055513
https://www.qcrao.com/2019/04/25/dive-into-go-interface/

https://www.cnblogs.com/landv/p/11589074.html

https://github.com/cch123/asmshare/blob/master/layout.md

Go 语言在编译期间对代码进行类型检查，上述代码总共触发了三次类型检查：
1.将变量赋值给接口类型
2.变量通过函数调用，函数参数是接口类型
3.变量从函数返回，函数返回参数是接口类型

```
//汇编
Go 汇编使用的是caller-save模式，被调用函数的入参参数、返回值都由调用者维护、准备。因此，当需要调用一个函数时，需要先将这些工作准备好，才调用下一个函数，另外这些都需要进行内存对齐，对齐的大小是 sizeof(uintptr)。
函数参数通过栈传递

有4个核心的伪寄存器，这4个寄存器是编译器用来维护上下文、特殊标识等作用的：
FP(Frame pointer): arguments and locals
使用如 symbol+offset(FP)的方式，引用 callee 函数的入参参数。例如 arg0+0(FP)，arg1+8(FP)，使用 FP 必须加 symbol ，否则无法通过编译(从汇编层面来看，symbol 没有什么用，加 symbol 主要是为了提升代码可读性)。另外，需要注意的是：往往在编写 go 汇编代码时，要站在 callee 的角度来看(FP)，在 callee 看来，(FP)指向的是 caller 调用 callee 时传递的第一个参数的位置。假如当前的 callee 函数是 add，在 add 的代码中引用 FP，该 FP 指向的位置不在 callee 的 stack frame 之内。而是在 caller 的 stack frame 上，指向调用 add 函数时传递的第一个参数的位置，经常在 callee 中用symbol+offset(FP)来获取入参的参数值。

PC(Program counter): jumps and branches
实际上就是在体系结构的知识中常见的 pc 寄存器，在 x86 平台下对应 ip 寄存器，amd64 上则是 rip。除了个别跳转之外，手写 plan9 汇编代码时，很少用到 PC 寄存器

SB(Static base pointer): global symbols
全局静态基指针，一般用在声明函数、全局变量中。

SP(Stack pointer): top of stack
该寄存器也是最具有迷惑性的寄存器，因为会有伪 SP 寄存器和硬件 SP 寄存器之分。plan9 的这个伪 SP 寄存器指向当前栈帧第一个局部变量的结束位置(为什么说是结束位置，可以看下面寄存器内存布局图)，使用形如 symbol+offset(SP) 的方式，引用函数的局部变量。offset 的合法取值是 [-framesize, 0)，注意是个左闭右开的区间。假如局部变量都是 8 字节，那么第一个局部变量就可以用 localvar0-8(SP) 来表示。与硬件寄存器 SP 是两个不同的东西，在栈帧 size 为 0 的情况下，伪寄存器 SP 和硬件寄存器 SP 指向同一位置。手写汇编代码时，如果是 symbol+offset(SP)形式，则表示伪寄存器 SP。如果是 offset(SP)则表示硬件寄存器 SP。务必注意：对于编译输出(go tool compile -S / go tool objdump)的代码来讲，所有的 SP 都是硬件 SP 寄存器，无论是否带 symbol（这一点非常具有迷惑性，需要慢慢理解。往往在分析编译输出的汇编时，看到的就是硬件 SP 寄存器）。


所有用户空间的数据都可以通过FP/SP(局部数据、输入参数、返回值)和SB(全局数据)访问。 通常情况下，不会对SB/FP寄存器进行运算操作，通常情况以会以SB/FP/SP作为基准地址，进行偏移解引用 等操作。

在plan9汇编里还可以直接使用的amd64的通用寄存器，应用代码层面会用到的通用寄存器主要是: rax, rbx, rcx, rdx, rdi, rsi, r8~r15这14个寄存器。plan9中使用寄存器不需要带r或e的前缀，例如rax，只要写AX即可:
```

<img src="..\images\stack.png" alt="stack" style="zoom:67%;" />

```go
func main() {
	c := "dasdasdsad2222"
	i(c)
}

func i(v interface{}) {

}


	0x0016 00022 (interface.go:3)	SUBQ	$56, SP
	0x001a 00026 (interface.go:3)	MOVQ	BP, 48(SP)
	0x001f 00031 (interface.go:3)	LEAQ	48(SP), BP
	0x0024 00036 (interface.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0024 00036 (interface.go:3)	FUNCDATA	$1, gclocals·ff19ed39bdde8a01a800918ac3ef0ec7(SB)
	0x0024 00036 (interface.go:3)	FUNCDATA	$3, "".main.stkobj(SB)
	0x0024 00036 (interface.go:4)	LEAQ	go.string."dasdasdsad2222"(SB), AX
	0x002b 00043 (interface.go:4)	MOVQ	AX, "".c+16(SP)			//字符串地址     拼装StringHeader结构
	0x0030 00048 (interface.go:4)	MOVQ	$14, "".c+24(SP)        //字符串长度
	0x0039 00057 (interface.go:5)	MOVQ	AX, ""..autotmp_1+32(SP) 
	0x003e 00062 (interface.go:5)	MOVQ	$14, ""..autotmp_1+40(SP)
	0x0047 00071 (interface.go:5)	LEAQ	type.string(SB), AX    // string的_type
	0x004e 00078 (interface.go:5)	MOVQ	AX, (SP)				//栈顶位置写入_type
	0x0052 00082 (interface.go:5)	LEAQ	""..autotmp_1+32(SP), AX // eface的data 
	0x0057 00087 (interface.go:5)	MOVQ	AX, 8(SP) 				// 栈顶+8写入data 
	0x005c 00092 (interface.go:5)	PCDATA	$1, $0
	0x005c 00092 (interface.go:5)	NOP
	0x0060 00096 (interface.go:5)	CALL	"".i(SB)
	0x0065 00101 (interface.go:6)	MOVQ	48(SP), BP
	0x006a 00106 (interface.go:6)	ADDQ	$56, SP
	0x006e 00110 (interface.go:6)	RET
```



```go
//无接口
type eface struct { // 16 字节
	_type *_type
	data  unsafe.Pointer
}

type iface struct { // 16 字节
	tab  *itab
	data unsafe.Pointer
}

type itab struct { // 32 字节
	inter *interfacetype //  接口元信息
	_type *_type
	hash  uint32      //hash 是对 _type.hash 当我们想将 interface 类型转换成具体类型时，可以使用该字段快速判断目标类型和具体类型 runtime._type 是否一致
	_     [4]byte
	fun   [1]uintptr  //fun 是一个动态大小的数组，它是一个用于动态派发的虚函数表，存储了一组函数指针。虽然该变量被声明成大小固定的数组，但是在使用时会通过原始指针获取其中的数据，所以 fun 数组中保存的元素数量是不确定的；
}

type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}


//下面的这些方法是将指定的类型转换成interface类型，但是下面的这些方法返回的仅仅是返回data指针

// 转换对象成一个 interface{}
func convT2E(t *_type, elem unsafe.Pointer) (e eface)
// 转换uint16成一个interface的data指针
func convT16(val uint16) (x unsafe.Pointer)
// 转换uint32成一个interface的data指针
func convT32(val uint32) (x unsafe.Pointer)
// 转换uint64成一个interface的data指针
func convT64(val uint64) (x unsafe.Pointer)
// 转换string成一个interface的data指针
func convTstring(val string) (x unsafe.Pointer)
// 转换slice成一个interface的data指针
func convTslice(val []byte) (x unsafe.Pointer)
// 转换t类型的元素到interface{}, 这里的t不是指针类型
func convT2Enoptr(t *_type, elem unsafe.Pointer) (e eface)
// 指定类型的到 interface 的转换
func convT2I(tab *itab, elem unsafe.Pointer) (i iface)
// 指定类型到 interface 的转换，不是指针
func convT2Inoptr(tab *itab, elem unsafe.Pointer) (i iface)
// interface到interface的转换。
func convI2I(inter *interfacetype, i iface) (r iface)

// Needs to be in sync with ../cmd/link/internal/ld/decodesym.go:/^func.commonsize,
// ../cmd/compile/internal/gc/reflect.go:/^func.dcommontype and
// ../reflect/type.go:/^type.rtype.
// ../internal/reflectlite/type.go:/^type.rtype.
type _type struct {
	size       uintptr          //字段存储了类型占用的内存空间，为内存空间的分配提供信息；
	ptrdata    uintptr
	hash       uint32           //字段能够帮助我们快速确定类型是否相等；
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	equal      func(unsafe.Pointer, unsafe.Pointer) bool   //字段用于判断当前类型的多个对象是否相等，该字段是为了减少 Go 语言二进制包大小从 typeAlg 结构体中迁移过来的
	gcdata     *byte
	str        nameOff
	ptrToThis  typeOff
}

func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
	x := mallocgc(t.size, t, true)
	// TODO: We allocate a zeroed object only to overwrite it with actual data.
	// Figure out how to avoid zeroing. Also below in convT2Eslice, convT2I, convT2Islice.
	typedmemmove(t, x, elem)
	e._type = t
	e.data = x
	return
}

func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
    //原值类型
	t := tab._type
    //堆上申请内存
	x := mallocgc(t.size, t, true)
    //复制到堆上
	typedmemmove(t, x, elem)
	i.tab = tab
	i.data = x
	return
}


func assertI2I(inter *interfacetype, i iface) (r iface) {
	tab := i.tab
	if tab == nil {
		// explicit conversions require non-nil interface value.
		panic(&TypeAssertionError{nil, nil, &inter.typ, ""})
	}
	if tab.inter == inter {
		r.tab = tab
		r.data = i.data
		return
	}
	r.tab = getitab(inter, tab._type, false)
	r.data = i.data
	return
}


//类型转换异常
type Student struct {
	Name string
	Age  int
}

func main() {
	var i interface{} = new(Student)
	s := i.(Student)
	fmt.Println(s)
}

	0x005a 00090 (i5.go:11)	LEAQ	type.*"".Student(SB), CX
	0x0061 00097 (i5.go:11)	MOVQ	CX, "".i+64(SP)
	0x0066 00102 (i5.go:11)	MOVQ	AX, "".i+72(SP)
	0x006b 00107 (i5.go:12)	MOVQ	$0, ""..autotmp_3+168(SP)
	0x0077 00119 (i5.go:12)	XORPS	X0, X0
	0x007a 00122 (i5.go:12)	MOVUPS	X0, ""..autotmp_3+176(SP)
	0x0082 00130 (i5.go:12)	MOVQ	"".i+72(SP), AX
	0x0087 00135 (i5.go:12)	MOVQ	"".i+64(SP), CX
	0x008c 00140 (i5.go:12)	LEAQ	type."".Student(SB), DX
	0x0093 00147 (i5.go:12)	CMPQ	DX, CX
	0x0096 00150 (i5.go:12)	JEQ	157
	0x0098 00152 (i5.go:12)	JMP	423
	。。。。。。
	0x01a7 00423 (i5.go:12)	PCDATA	$0, $-1
	0x01a7 00423 (i5.go:12)	MOVQ	CX, (SP)
	0x01ab 00427 (i5.go:12)	MOVQ	DX, 8(SP)
	0x01b0 00432 (i5.go:12)	LEAQ	type.interface {}(SB), AX
	0x01b7 00439 (i5.go:12)	MOVQ	AX, 16(SP)
	0x01bc 00444 (i5.go:12)	NOP
	0x01c0 00448 (i5.go:12)	CALL	runtime.panicdottypeE(SB)

// panicdottypeE is called when doing an e.(T) conversion and the conversion fails.
// have = the dynamic type we have.
// want = the static type we're trying to convert to.
// iface = the static type we're converting from.
func panicdottypeE(have, want, iface *_type) {
	panic(&TypeAssertionError{iface, have, want, ""})
}


//接口转接口
func convI2I(inter *interfacetype, i iface) (r iface) {
	//inter要转换的接口
    //i目前实现的接口
    tab := i.tab
	if tab == nil {
		return
	}
    //如果相同复制指针
	if tab.inter == inter {
		r.tab = tab
		r.data = i.data
		return
	}
    
	r.tab = getitab(inter, tab._type, false)
	r.data = i.data
	return
}
//inter待转换的类型
//typ实际的数据类型
//canfail是否可以转换失败
func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
    //无效的inter
	if len(inter.mhdr) == 0 {
		throw("internal error - misuse of itab")
	}
	// 判断type的标志位 （不清楚这标志位有啥用）
	if typ.tflag&tflagUncommon == 0 {
		if canfail {
			return nil
		}
		name := inter.typ.nameOff(inter.mhdr[0].name)
		panic(&TypeAssertionError{nil, typ, &inter.typ, name.name()})
	}

	var m *itab
	// 首先查看已经存在的表查找我们需要的
    // 这是大多数通常情况，不需要上锁
    // 使用原子操作确保我们看到线程以前锁做的任何操作
	t := (*itabTableType)(atomic.Loadp(unsafe.Pointer(&itabTable)))
	if m = t.find(inter, typ); m != nil {
		//找到直接结束流程
        goto finish
	}
	// 未找到上锁
	lock(&itabLock)
    // 当前表查找
	if m = itabTable.find(inter, typ); m != nil {
		unlock(&itabLock)
		goto finish
	}
	
	// entry不存在创建一个新的
	m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*sys.PtrSize, 0, &memstats.other_sys))
	m.inter = inter
	m._type = typ
    // hash用来做类型转换，然后编译器静态的生成itab的hash，动态生成的itab's不参与类型转换
    // 注意m.hash 不是运行时itabTable用的hash
	
	m.hash = 0
	m.init()
	itabAdd(m)
	unlock(&itabLock)
finish:
	if m.fun[0] != 0 {
		return m
	}
	if canfail {
		return nil
	}
	// this can only happen if the conversion
	// was already done once using the , ok form
	// and we have a cached negative result.
	// The cached result doesn't record which
	// interface function was missing, so initialize
	// the itab again to get the missing function name.
	panic(&TypeAssertionError{concrete: typ, asserted: &inter.typ, missingMethod: m.init()})
}


unsafe.Point
func main() {

	var a = 10
	c := unsafe.Pointer(&a)
	ca(*(*int)(c))
}
func ca(int) {
}
	0x0024 00036 (i7.go:9)	MOVQ	$10, "".a+8(SP)         //栈顶+8 a = 10
	0x002d 00045 (i7.go:10)	LEAQ	"".a+8(SP), AX          //AX = &a
	0x0032 00050 (i7.go:10)	MOVQ	AX, "".c+24(SP)         //c = &a
	0x0037 00055 (i7.go:11)	TESTB	AL, (AX)                // ?
	0x0039 00057 (i7.go:11)	MOVQ	"".a+8(SP), AX          // AX = 10
	0x003e 00062 (i7.go:11)	MOVQ	AX, ""..autotmp_2+16(SP) // 10拷贝到栈顶+16
	0x0043 00067 (i7.go:11)	MOVQ	AX, (SP)              //10拷贝到栈顶
	0x0047 00071 (i7.go:11)	PCDATA	$1, $0
	0x0047 00071 (i7.go:11)	CALL	"".ca(SB)              //调用函数
	0x004c 00076 (i7.go:12)	MOVQ	32(SP), BP

```

