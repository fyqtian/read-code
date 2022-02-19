### array

无论是在栈上还是静态存储区，数组在内存中都是一连串的内存空间，**我们通过指向数组开头的指针**、

```go
// 如果数组元素 <= 4个 存放在栈上 否在变量就会在静态存储区初始化然后拷贝到栈上
func main() {
   a := [3]int{1, 2, 3}
   c(a)
}
func c([3]int) {

}

	0X0024 00036 (main.go:4)	MOVQ	$0, "".a+24(SP)
	0x002d 00045 (main.go:4)	XORPS	X0, X0
	0x0030 00048 (main.go:4)	MOVUPS	X0, "".a+32(SP)
	0x0035 00053 (main.go:4)	MOVQ	$1, "".a+24(SP)  
	0x003e 00062 (main.go:4)	MOVQ	$2, "".a+32(SP)
	0x0047 00071 (main.go:4)	MOVQ	$3, "".a+40(SP)
	0x0050 00080 (main.go:5)	MOVQ	$1, (SP)
	0x0058 00088 (main.go:5)	MOVQ	$2, 8(SP)
	0x0061 00097 (main.go:5)	MOVQ	$3, 16(SP)
	0x006a 00106 (main.go:5)	PCDATA	$1, $0
	0x006a 00106 (main.go:5)	CALL	"".c(SB)
	0x006f 00111 (main.go:6)	MOVQ	48(SP), BP



func main() {
	arr := d()
	arr[100] = 123456
}

func d() (rs [8912]int) {
	return
}
	0x0030 00048 (a3.go:3)	SUBQ	$142600, SP       // 申请栈空间
	0x0037 00055 (a3.go:3)	MOVQ	BP, 142592(SP)    // 栈顶
	0x003f 00063 (a3.go:3)	LEAQ	142592(SP), BP    // 
	0x0047 00071 (a3.go:4)	CALL	"".d(SB)
	0x004c 00076 (a3.go:4)	LEAQ	"".arr+71296(SP), DI //返回的地址
	0x0054 00084 (a3.go:4)	MOVQ	SP, SI
	0x0057 00087 (a3.go:4)	MOVL	$8912, CX      
	0x005c 00092 (a3.go:4)	REP
	0x005d 00093 (a3.go:4)	MOVSQ
	0x005f 00095 (a3.go:5)	MOVQ	$123456, "".arr+72096(SP) arr[100] = 123456
	0x006b 00107 (a3.go:6)	MOVQ	142592(SP), BP
	0x0073 00115 (a3.go:6)	ADDQ	$142600, SP


func main() {
	m := [89120]int{}
	m[100] = 7788787
}
0x0030 00048 (a4.go:3)	SUBQ	$712968, SP
	0x0037 00055 (a4.go:3)	MOVQ	BP, 712960(SP)
	0x003f 00063 (a4.go:3)	LEAQ	712960(SP), BP
	0x0047 00071 (a4.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0047 00071 (a4.go:3)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0047 00071 (a4.go:4)	LEAQ	"".m(SP), DI
	0x004b 00075 (a4.go:4)	MOVL	$89120, CX
	0x0050 00080 (a4.go:4)	XORL	AX, AX
	0x0052 00082 (a4.go:4)	REP
	0x0053 00083 (a4.go:4)	STOSQ
	0x0055 00085 (a4.go:5)	MOVQ	$7788787, "".m+800(SP)
	0x0061 00097 (a4.go:6)	MOVQ	712960(SP), BP
	0x0069 00105 (a4.go:6)	ADDQ	$712968, SP
	0x0070 00112 (a4.go:6)	RET


如果两个数组比较会调用
// in asm_*.s
//go:noescape
func memequal(a, b unsafe.Pointer, size uintptr) bool
```

