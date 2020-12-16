### bytes

**buffer**

```go
//example
func main() {
   b := make([]byte, 0, 16)
   buf := bytes.NewBuffer(b)
   buf.WriteString("abc")
   b = append(b, 'd')
   fmt.Println(buf.String())
}
//output
dbc
```

```go

//一个Buffer是一个可变大小的缓冲具有读和写方法
//Buffer 的 零值 是一个 空的 buffer，但是可以使用
type Buffer struct {
   buf      []byte // 缓冲
   off      int    // 读的位置下标
   lastRead readOp // 上次读取操作
}


//NewBuffer创建并初始化一个新的Buffer,新的Buffer拥有自己的buf,调用者不应该使用这个buf
//NewBuffer打算准备一个BUffer读已经存在的数据，它也能够初始化缓冲大小用于写
//为了做到这点，buf应该拥有容量，但是长度为0
func NewBuffer(buf []byte) *Buffer { return &Buffer{buf: buf} }


// WriteString appends the contents of s to the buffer, growing the buffer as
// needed. The return value n is the length of s; err is always nil. If the
// buffer becomes too large, WriteString will panic with ErrTooLarge.
func (b *Buffer) WriteString(s string) (n int, err error) {
    //变更为未读取
	b.lastRead = opInvalid
    //判断是否需要扩容
	m, ok := b.tryGrowByReslice(len(s))
	if !ok {
        //扩容
       
		m = b.grow(len(s))
	}
    //copy数据
	return copy(b.buf[m:], s), nil
}

//tryGrowByReslice是一个内联版本的切片扩容
//返回被写入的下标位置
func (b *Buffer) tryGrowByReslice(n int) (int, bool) {
    //如果当前长度+待写入长度<=容量 不需要扩容
	if l := len(b.buf); n <= cap(b.buf)-l {
		b.buf = b.buf[:l+n]
		return l, true
	}
	return 0, false
}

//grow为了保证缓冲区空间足够存放n个bytes扩容缓冲
//返回写入位置的下标
//如果不能扩容缓冲将会painc ErrTooLarge
func (b *Buffer) grow(n int) int {
	m := b.Len()
	// If buffer is empty, reset to recover space.
    //如果长度为0 并且读取的位置不为0
	if m == 0 && b.off != 0 {
		b.Reset()
	}
	//尝试当前buf是否满足扩容需求
	if i, ok := b.tryGrowByReslice(n); ok {
		return i
	}
    //如果buf==nil并且申请的n<=64直接make 64长度的切片
	if b.buf == nil && n <= smallBufferSize {
		b.buf = make([]byte, n, smallBufferSize)
		return 0
	}
	c := cap(b.buf)
    //如果申请的长度<=当前容量一半减去已经分配的长度
	if n <= c/2-m {
        //当前切片剩余空间够用，复用当前的切片
		copy(b.buf, b.buf[b.off:])
	} else if c > maxInt-c-n {
        //认为是非法的申请
		panic(ErrTooLarge)
	} else {
        //剩余空间不够 重新分配空间
		buf := makeSlice(2*c + n)
        //未读取的旧数据copy到新空间
		copy(buf, b.buf[b.off:])
		b.buf = buf
	}
	// Restore b.off and len(b.buf).
	b.off = 0
	b.buf = b.buf[:m+n]
	return m
}

//Reset重置buffer为empty
//但是保留底切片
//Reset等同于Truncate(0)
func (b *Buffer) Reset() {
	b.buf = b.buf[:0]
	b.off = 0
	b.lastRead = opInvalid
}
```

