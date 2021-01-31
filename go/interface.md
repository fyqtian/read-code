### interface

//汇编
https://www.cnblogs.com/yjf512/p/6132868.html
https://blog.csdn.net/tencent_teg/article/details/108413692
https://zhuanlan.zhihu.com/p/56750445
https://blog.csdn.net/u010853261/article/details/101060546
https://www.codercto.com/a/103852.html
https://zhuanlan.zhihu.com/p/27055513
https://www.qcrao.com/2019/04/25/dive-into-go-interface/
Go 语言在编译期间对代码进行类型检查，上述代码总共触发了三次类型检查：
1.将变量赋值给接口类型
2.变量通过函数调用，函数参数是接口类型
3.变量从函数返回，函数返回参数是接口类型

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
	// inter+_type表示类型
	inter *interfacetype
	_type *_type
	hash  uint32      //hash 是对 _type.hash 当我们想将 interface 类型转换成具体类型时，可以使用该字段快速判断目标类型和具体类型 runtime._type 是否一致
	_     [4]byte
	fun   [1]uintptr  //fun 是一个动态大小的数组，它是一个用于动态派发的虚函数表，存储了一组函数指针。虽然该变量被声明成大小固定的数组，但是在使用时会通过原始指针获取其中的数据，所以 fun 数组中保存的元素数量是不确定的；
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