### 内存逃逸分析

https://blog.csdn.net/qq_35587463/article/details/104221280

go 在**编译阶段**确立逃逸，注意并不是在运行时



其实就是为了尽可能在**栈上分配内存**，我们可以反过来想，如果变量都分配到堆上了会出现什么事情？例如：

- 垃圾回收（GC）的压力不断增大

- 申请、分配、回收内存的系统开销增大（相对于栈）

- 动态分配产生一定量的内存碎片

  

其实总的来说，就是频繁申请、分配堆内存是有一定 “代价” 的。会影响应用程序运行的效率，间接影响到整体系统。因此 “按需分配” 最大限度的灵活利用资源，才是正确的治理之道。这就是为什么需要逃逸分析的原因，你觉得呢？

```go
type User struct {
   ID     int64
   Name   string
   Avatar string
}
func GetUserInfo() *User {
   return &User{
      ID:     666666,
      Name:   "sim lou",
      Avatar: "https://www.baidu.com/avatar/666666",
   }
}
func getArray() (rs [10]int) {
	return
}

func main() {
	_ = GetUserInfo()
	_ = getArray()
}
go build -gcflags '-N-m -l' memory-leak.go
//output
.memory-leak.go:10:9: &User literal escapes to heap



main()
	0x0016 00022 (memory-leak.go:20)	SUBQ	$88, SP
	0x001a 00026 (memory-leak.go:20)	MOVQ	BP, 80(SP)
	0x001f 00031 (memory-leak.go:20)	LEAQ	80(SP), BP
	0x0024 00036 (memory-leak.go:20)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0024 00036 (memory-leak.go:20)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0024 00036 (memory-leak.go:21)	PCDATA	$1, $0
	0x0024 00036 (memory-leak.go:21)	CALL	"".GetUserInfo(SB)
	0x0029 00041 (memory-leak.go:22)	CALL	"".getArray(SB)

GetUserInfo()
	0x001a 00026 (memory-leak.go:9)	SUBQ	$32, SP
	0x001e 00030 (memory-leak.go:9)	MOVQ	BP, 24(SP)
	0x0023 00035 (memory-leak.go:9)	LEAQ	24(SP), BP
	0x0028 00040 (memory-leak.go:9)	MOVQ	$0, "".~r0+40(SP)
	0x0031 00049 (memory-leak.go:10)	LEAQ	type."".User(SB), AX     // AX = &User{}
	0x0038 00056 (memory-leak.go:10)	MOVQ	AX, (SP)               
	0x0040 00064 (memory-leak.go:10)	CALL	runtime.newobject(SB)   
	0x0045 00069 (memory-leak.go:10)	MOVQ	8(SP), AX                // AX = runtime.newobject
	0x004a 00074 (memory-leak.go:10)	MOVQ	AX, ""..autotmp_1+16(SP) // 函数返回值地址
	0x004f 00079 (memory-leak.go:11)	MOVQ	$666666, (AX)            //User.ID = 66666
	0x0056 00086 (memory-leak.go:10)	MOVQ	""..autotmp_1+16(SP), AX 
	0x005b 00091 (memory-leak.go:10)	TESTB	AL, (AX)
	0x005d 00093 (memory-leak.go:12)	MOVQ	$7, 16(AX)               //User.Name.Len = 7
	0x0065 00101 (memory-leak.go:12)	LEAQ	8(AX), DI
	0x0069 00105 (memory-leak.go:12)	CMPL	runtime.writeBarrier(SB), $0
	0x0070 00112 (memory-leak.go:12)	JEQ	116
	0x0072 00114 (memory-leak.go:12)	JMP	209
	0x0074 00116 (memory-leak.go:12)	LEAQ	go.string."sim lou"(SB), CX
	0x007b 00123 (memory-leak.go:12)	MOVQ	CX, 8(AX)                //User.Name.Data = &"sim lou"
	0x007f 00127 (memory-leak.go:12)	NOP
	0x0080 00128 (memory-leak.go:12)	JMP	130
	0x0082 00130 (memory-leak.go:10)	PCDATA	$0, $-1
	0x0082 00130 (memory-leak.go:10)	MOVQ	""..autotmp_1+16(SP), AX
	0x0087 00135 (memory-leak.go:10)	TESTB	AL, (AX) 
	0x0089 00137 (memory-leak.go:13)	MOVQ	$35, 32(AX)             //User.Avatar.Len = 35 
	0x0091 00145 (memory-leak.go:13)	LEAQ	24(AX), DI
	0x0095 00149 (memory-leak.go:13)	PCDATA	$0, $-2
	0x0095 00149 (memory-leak.go:13)	CMPL	runtime.writeBarrier(SB), $0
	0x009c 00156 (memory-leak.go:13)	JEQ	162
	0x009e 00158 (memory-leak.go:13)	NOP
	0x00a0 00160 (memory-leak.go:13)	JMP	195
	0x00a2 00162 (memory-leak.go:13)	LEAQ	go.string."https://www.baidu.com/avatar/666666"(SB), CX
	0x00a9 00169 (memory-leak.go:13)	MOVQ	CX, 24(AX)                //User.Avatar.Data = CX 
	0x00ad 00173 (memory-leak.go:13)	JMP	175
	0x00af 00175 (memory-leak.go:10)	PCDATA	$0, $-1
	0x00af 00175 (memory-leak.go:10)	MOVQ	""..autotmp_1+16(SP), AX
	0x00b4 00180 (memory-leak.go:10)	MOVQ	AX, "".~r0+40(SP)
	0x00b9 00185 (memory-leak.go:10)	MOVQ	24(SP), BP
	0x00be 00190 (memory-leak.go:10)	ADDQ	$32, SP
	0x00c2 00194 (memory-leak.go:10)	RET




func main() {
	fmt.Println("abc")
}
//output
memory-leak.go:30:14: "abc" escapes to heap
reflect.Value 会导致变量分配在heap上

// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}
	
	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i)
	return unpackEface(i)
}

// Dummy annotation marking that the value x escapes,
// for use in cases where the reflect code is so clever that
// the compiler cannot follow.
// dummy注解标记x逃逸，在反射代码的这个例子中编译器不会跟随？
func escapes(x interface{}) {
	if dummy.b {
		dummy.x = x
	}
}


//对interface推断不出类型的 好像要分配到堆上

func main() {
	dummy.x = 100
	dummy.age = 11
}

var dummy struct {
	x   interface{}
	age int
}
//output
.memory-leak2.go:4:10: 100 escapes to heap

```