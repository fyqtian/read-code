### array

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
```