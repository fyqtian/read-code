### strings

```go
//example
package main

import (
   "os"
   "strings"
)

func main() {
   f := strings.NewReader("abc")
   f.WriteTo(os.Stdout)

   //body, err := ioutil.ReadAll(f)
   //fmt.Println(body, err)
}
```



```go
// Reader实现io.Reader, io.ReaderAt, io.Seeker, io.WriterTo,
// io.ByteScanner, and io.RuneScanner interfaces
// The zero value for Reader operates like a Reader of an empty string.
type Reader struct {
	s        string
	i        int64 // 当前读到的位置
	prevRune int   // index of previous rune; or < 0
}

// NewReader 一个从s读取的Reader指针
// 和bytes.NewBufferString相似，但效率更高并且只能读.
func NewReader(s string) *Reader { return &Reader{s, 0, -1} }

//返回未读取的长度
func (r *Reader) Len() int {
    //如果已经读到结束返回0
	if r.i >= int64(len(r.s)) {
		return 0
	}
	return int(int64(len(r.s)) - r.i)
}

//Size 返回原始数据的长度
//Size是ReadAt可读的长度
//不受其他方法影响
func (r *Reader) Size() int64 { return int64(len(r.s)) }


func (r *Reader) Read(b []byte) (n int, err error) {
    //如果已经读到结束 返回EOF
	if r.i >= int64(len(r.s)) {
		return 0, io.EOF
	}
	r.prevRune = -1
    //从源数据中拷贝数据到切片
	n = copy(b, r.s[r.i:])
    //更新r.i
	r.i += int64(n)
	return
}
//从off的位置开始读
func (r *Reader) ReadAt(b []byte, off int64) (n int, err error) {
	// cannot modify state - see io.ReaderAt
	if off < 0 {
		return 0, errors.New("strings.Reader.ReadAt: negative offset")
	}
	if off >= int64(len(r.s)) {
		return 0, io.EOF
	}
    //如果b比读取的数据大 返回io.EOF
	n = copy(b, r.s[off:])
	if n < len(b) {
		err = io.EOF
	}
	return
}
//r.i回退一位
func (r *Reader) UnreadByte() error {
	if r.i <= 0 {
		return errors.New("strings.Reader.UnreadByte: at beginning of string")
	}
	r.prevRune = -1
	r.i--
	return nil
}

// WriteTo implements the io.WriterTo interface.
func (r *Reader) WriteTo(w io.Writer) (n int64, err error) {
	r.prevRune = -1
    //如果数据已经读完
	if r.i >= int64(len(r.s)) {
		return 0, nil
	}
    //找出未读的数据
	s := r.s[r.i:]
    //写入write接口
	m, err := io.WriteString(w, s)
	if m > len(s) {
		panic("strings.Reader.WriteTo: invalid WriteString count")
	}
    //更新r.i
	r.i += int64(m)
	n = int64(m)
	if m != len(s) && err == nil {
		err = io.ErrShortWrite
	}
	return
}

```