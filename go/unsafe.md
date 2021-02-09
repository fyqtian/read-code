### unsafe

https://louyuting.blog.csdn.net/article/details/100178972

golang原有指针

1. 指针是不能做数学运算。
2. 不同类型的指针不能相互转换。
3. 不同类型的指针不能使用 == 或者 != 比较。
4. 不同类型的指针变量不能相互赋值



uintptr：uintptr 是 Go 的**内置类型**。返回无符号整数，可存储一个完整的地址，后续常用于指针数学运算。

Pointer 允许程序破坏类型系统并**对任意的内存进行读写**。使用应非常小心。

unsafe 包用于 Go 编译器，**在编译阶段使用**。从名字就可以看出来，它是不安全的，官方并不建议使用。

但是高阶的 Gopher，怎么能不会使用 unsafe 包呢？它可以绕过 Go 语言的类型系统，直接操作内存。例如，一般我们不能操作一个结构体的未导出成员，但是通过 unsafe 包就能做到。unsafe 包让我可以直接读写内存，还管你什么导出还是未导出。





```go
// Pointer表示一个指针指向任意类型，有4个特别得操作只适用Pointer, 不适用其他类型
// 任意指针类型可以转换为pointer
// 一个Pointer可以被转换成任一其他类型得指针
// uintptr可以被转换成Pointer
// Pointer可以转换成uintptr

// 因此，Pointer允许程序破坏类型系统并进行读写
// 下列模式调用指针是允许得
// 代码不使用这些模式可能是非法得或在将来是非法得
// 即使下面合法得模式也有重要得警告

// 运行go ver可以帮助找到使用不符合模式得指针，但是不能保证代码是合法得
//
// (1) Conversion of a *T1 to Pointer to *T2.
// 前提T2不大于T1并且享有相同得内存布局，这个转换允许将一种类型得数据解析为另外一个类型
//
//	func Float64bits(f float64) uint64 {
//		return *(*uint64)(unsafe.Pointer(&f))
//	}
//
// (2) Conversion of a Pointer to a uintptr (but not back to Pointer).
// 转换Pointer到uintptr产生一个integer表示value得地址，通常使用uintptr用来打印

// uintptr转换成Pointer通常来说是无效得
// uintptr是integer,不是一个引用
// 将指针转换为uintptr将创建一个整数值，将没有指针语义
// 即使uintprt持有对象得地址，垃圾回收期将不会更新uintprt得值，如果对象移动，uintptr也不会保留这个对象

// 剩下得模式只有枚举转换有效
// (3) Conversion of a Pointer to a uintptr and back, with arithmetic.
// 如果p指向分配得对象，它可以通过转换成uintptr，通过添加偏移量转换回Pointer
//	p = unsafe.Pointer(uintptr(p) + offset)
//
// 大多数通常使用这种模式访问struct的字段或者数组的元素
//	// equivalent to f := unsafe.Pointer(&s.f)
//	f := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f))
//
//	// equivalent to e := unsafe.Pointer(&x[i])
//	e := unsafe.Pointer(uintptr(unsafe.Pointer(&x[0])) + i*unsafe.Sizeof(x[0]))
//
// 这是合法的对指针进行加法或减法。使用&^对指针取整也是合法的,通常为了对齐
// 在所有的例子中，结果必须继续指向原始分配的对象
// 与C语言不同的是，将指针移到末尾之后是无效的，其原始分配：

//	// INVALID: end points outside allocated space.
//	var s thing
//	end = unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Sizeof(s))

//	// INVALID: end points outside allocated space.
//	b := make([]byte, n)
//	end = unsafe.Pointer(uintptr(unsafe.Pointer(&b[0])) + uintptr(n))
//
// 注意两种转换必须出现在同一表达式，只有他们之间的中间运算
//  // 在转换回Pointer之前非法 uintptr不能存储在变量中
//	u := uintptr(p)
//	p = unsafe.Pointer(u + offset)
//
// 注意pointr必须指向分配的内存,所以可能不是Nil
// Note that the pointer must point into an allocated object, so it may not be nil.
//
//	// INVALID: conversion of nil pointer
//	u := unsafe.Pointer(nil)
//	p := unsafe.Pointer(uintptr(u) + offset)
//
// 调用syscall.Syscall需要将Pointer转成uintptr
// (4) Conversion of a Pointer to a uintptr when calling syscall.Syscall.
// 系统调用方法在syscall包直接传递uintptr参数到操作系统，根据调用的一些具体情况，把他们当成pointer解释
// 也就是说系统调用实现隐式的转换某些参数uintptr到pointer
//
//	syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))
// 
// The compiler handles a Pointer converted to a uintptr in the argument list of
// a call to a function implemented in assembly by arranging that the referenced
// allocated object, if any, is retained and not moved until the call completes,
// even though from the types alone it would appear that the object is no longer
// needed during the call.
//
// 为了让编译器识别这个模式，转换必须出现在参数列表中
// 
//	// INVALID: uintptr cannot be stored in variable
//	// before implicit conversion back to Pointer during system call.
//	u := uintptr(unsafe.Pointer(p))
//	syscall.Syscall(SYS_READ, uintptr(fd), u, uintptr(n))
//
// 转换reflect.Value.Point或者reflect.Value.UnsafeAdd从uintptr到Pointer
// (5) Conversion of the result of reflect.Value.Pointer or reflect.Value.UnsafeAddr
// from uintptr to Pointer.
// reflect包的Value结构方法Pointed和UnsafeAddr返回uintptr而不是unsafe.Point为了保持调用者修改结果到任意类型，
// 这意味着结果是容易变化的，必须立即转换成Pointer
//	p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
//
//	// INVALID: uintptr cannot be stored in variable
//	// before conversion back to Pointer.
//	u := reflect.ValueOf(new(int)).Pointer()
//	p := (*int)(unsafe.Pointer(u))
//
// 转换 Pointer到reflect.SliceHeader或者reflect.StringHeader
// (6) Conversion of a reflect.SliceHeader or reflect.StringHeader Data field to or from Pointer.
// 在先前的列子中，reflect结构SliceHeader和SliceHeader申明Data字段为uintptr为了调用者修改结果到Pointer第一时间引入
// "unsafe",然而这意味着SliceHeader和StringHeader仅当解释为slice或者string合法
/
//	var s string
//	hdr := (*reflect.StringHeader)(unsafe.Pointer(&s)) // case 1
//	hdr.Data = uintptr(unsafe.Pointer(p))              // case 6 (this case)
//	hdr.Len = n
//
// 在这个用法中hr.Data 数据实际上是指字符串头中的指针，而不是uintptr变量本身。
// In this usage hdr.Data is really an alternate way to refer to the underlying
// pointer in the string header, not a uintptr variable itself.
//
// 通常，reflect.SliceHeader和reflect.StringHeader应该只用来 *reflect.SliceHeader和*reflect.StringHeader 	// // // 指向实际的slice或者字符串，而不是在项目中作为一个结构被申明或者分配变量

//	// INVALID: a directly-declared header will not hold Data as a reference.
//	var hdr reflect.StringHeader
//	hdr.Data = uintptr(unsafe.Pointer(p))
//	hdr.Len = n
//	s := *(*string)(unsafe.Pointer(&hdr)) // p possibly already lost
//
type Pointer *ArbitraryType
type ArbitraryType int

// Sizeof接收任意类型的参数并返回占用的bytes大小
// 假设变量v的形式类似var v = x
// 这个size不包含x引用的内存大小
// 例如如果x是一个切片，Sizeof返回slice描述的切片，而不是slice占用 的内存
func Sizeof(x ArbitraryType) uintptr
func main() {
	m := make([]int, 1024)
	fmt.Println(unsafe.Sizeof(m))
}
//output
24

// offsetof返回结构体中field表示的偏移量
//必须是结构体的filed，换句话说，他返回结构起始到字段的字节数
func Offsetof(x ArbitraryType) uintptr
func main() {
    type data struct {
		name string
		age  int
	}
	fmt.Println(unsafe.Offsetof(data{
		name: "",
		age:  0,
	}.age))
}
//output
16

//  返回 m，m 是指当类型进行内存对齐时，它分配到的内存地址能整除 m
// It is the same as the value returned by reflect.TypeOf(x).Align().
// As a special case, if a variable s is of struct type and f is a field
// within that struct, then Alignof(s.f) will return the required alignment
// of a field of that type within a struct. This case is the same as the
// value returned by reflect.TypeOf(s.f).FieldAlign().
// The return value of Alignof is a Go constant.
func Alignof(x ArbitraryType) uintptr

```

