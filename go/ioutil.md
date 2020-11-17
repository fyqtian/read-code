### ioutil

```go
//example
package main

import (
   "fmt"
   "io/ioutil"
   "strings"
)

func main() {
   f := strings.NewReader("abc")
   body, err := ioutil.ReadAll(f)
   fmt.Println(body, err)
}
```

```go
//ReadAll从r中读取数据直到发生err并返回已读的数据
//一次成功的读取返回err==nil，readall不会报告EOF错误
func ReadAll(r io.Reader) ([]byte, error) {
	//MinRead == 512
   return readAll(r, bytes.MinRead)
}


func readAll(r io.Reader, capacity int64) (b []byte, err error) {
	var buf bytes.Buffer

	//如果buffer溢出 返回 bytes.ErrTooLarge.
	//其他panic保持
	defer func() {
		e := recover()
		if e == nil {
			return
		}
		if panicErr, ok := e.(error); ok && panicErr == bytes.ErrTooLarge {
			err = panicErr
		} else {
			panic(e)
		}
	}()
	//没明白为啥
	//判断int是否溢出？
	if int64(int(capacity)) == capacity {
		buf.Grow(int(capacity))
	}
	_, err = buf.ReadFrom(r)
	return buf.Bytes(), err
}

// Grow grows the buffer's capacity, if necessary, to guarantee space for
// another n bytes. After Grow(n), at least n bytes can be written to the
// buffer without another allocation.
//Grow增加buffer的capcaity

//如果n是负数 panic.
// 如果buffer不能扩容 panic with ErrTooLarge.
func (b *Buffer) Grow(n int) {
	if n < 0 {
		panic("bytes.Buffer.Grow: negative count")
	}
	m := b.grow(n)
	b.buf = b.buf[:m]
}

// grow grows the buffer to guarantee space for n more bytes.
// It returns the index where bytes should be written.
// If the buffer can't grow it will panic with ErrTooLarge.
func (b *Buffer) grow(n int) int {
	m := b.Len()
	//如果buf是空的 reset
	if m == 0 && b.off != 0 {
		b.Reset()
	}
	// Try to grow by means of a reslice.
	if i, ok := b.tryGrowByReslice(n); ok {
		return i
	}
	//如果是第一次初始化b.buf==nil 并且n<=64 直接分配
	if b.buf == nil && n <= smallBufferSize {
		b.buf = make([]byte, n, smallBufferSize)
		return 0
	}
	c := cap(b.buf)
	if n <= c/2-m {
		// We can slide things down instead of allocating a new
		// slice. We only need m+n <= c to slide, but
		// we instead let capacity get twice as large so we
		// don't spend all our time copying.
		copy(b.buf, b.buf[b.off:])
	} else if c > maxInt-c-n {
		panic(ErrTooLarge)
	} else {
		// Not enough space anywhere, we need to allocate.
		buf := makeSlice(2*c + n)
		copy(buf, b.buf[b.off:])
		b.buf = buf
	}
	// Restore b.off and len(b.buf).
	b.off = 0
	b.buf = b.buf[:m+n]
	return m
}

// ReadFrom从r中读取数据直到结束并将读取的数据写入缓冲中，如必要会增加缓冲容量。
// 返回值n为从r读取并写入b的字节数；会返回读取时遇到的除了io.EOF之外的错误。
// 如果缓冲太大，ReadFrom会采用错误值ErrTooLarge引发panic。
func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error) {
	//opInvalid=0  Non-read operation.
    b.lastRead = opInvalid
	for {
		i := b.grow(MinRead)
		b.buf = b.buf[:i]
		m, e := r.Read(b.buf[i:cap(b.buf)])
		if m < 0 {
			panic(errNegativeRead)
		}

		b.buf = b.buf[:i+m]
		n += int64(m)
		if e == io.EOF {
			return n, nil // e is EOF, so return nil explicitly
		}
		if e != nil {
			return n, e
		}
	}
}

```

