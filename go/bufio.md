/src/bufio/bufio.go



bufio主要是使用自己的从底层读取数据存放在自己的buf中，后续的操作都从自己的buf中读取，减少io

```
//example
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
)
func main() {
	f, _ := os.Open("D:\\gocode\\test\\reader\\bufio.go")
	r := bufio.NewReaderSize(f, 16)
	for {
		val, isPrefix, err := r.ReadLine()
		fmt.Println(string(val), isPrefix, err)
		if err == io.EOF {
			break
		}
	}
}

```



```go
// Reader implements buffering for an io.Reader object.
type Reader struct {
	buf          []byte
	rd           io.Reader // reader provided by the client
	r, w         int       // buf read and write positions
	err          error
	lastByte     int // last byte read for UnreadByte; -1 means invalid
	lastRuneSize int // size of last rune read for UnreadRune; -1 means invalid
}

//常用的bufio.NewReader(rd io.Reader) *Reader
// defaultBufSize = 4096
func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)
}

//创建一个buf大小为size的*Reader
func NewReaderSize(rd io.Reader, size int) *Reader {
	//如果已经是bufio.Reader类型 并且rd.buf >= size
	b, ok := rd.(*Reader)
	if ok && len(b.buf) >= size {
		return b
	}
	//minReadBufferSize=16 size小于<16 就用默认
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	r := new(Reader)
	
	r.reset(make([]byte, size), rd)
	return r
}

func (b *Reader) reset(buf []byte, r io.Reader) {
    //不明白为什么要取址
	*b = Reader{
		buf:          buf,
		rd:           r,
		lastByte:     -1,
		lastRuneSize: -1,
	}
}


// 数据写入p中
// 返回写入p中的长度
// bytes从底层的reader读取
// 返回的n可能小于p的长度
// 如果需要景区的读取p的长度  使用io.ReadFull(b, p).
// At EOF, the count will be zero and err will be io.EOF.
func (b *Reader) Read(p []byte) (n int, err error) {
	n = len(p)
    //如果传入的p的长度为0
	if n == 0 {
        //如果有缓存但是传入的p==0 返回0，nil
		if b.Buffered() > 0 {
			return 0, nil
		}
        //如果p==nil或者长度==0 返回的0, err(如果是第一次读还是会返回nil)
		return 0, b.readErr()
	}
    //如果读取的位置等于写入的位置，再次进行reader的读取
	if b.r == b.w {
		if b.err != nil {
			return 0, b.readErr()
		}
        //如果p比buf大直接写入到p
		if len(p) >= len(b.buf) {
			// Large read, empty buffer.
			// Read directly into p to avoid copy.
			n, b.err = b.rd.Read(p)
			if n < 0 {
				panic(errNegativeRead)
			}
			if n > 0 {
                //记录最后个byte
				b.lastByte = int(p[n-1])
				b.lastRuneSize = -1
			}
			return n, b.readErr()
		}
		//第一次读取获取是已经读完缓冲区
		b.r = 0
		b.w = 0
        //数据读到buf中
		n, b.err = b.rd.Read(b.buf)
		if n < 0 {
			panic(errNegativeRead)
		}
		if n == 0 {
			return 0, b.readErr()
		}
        //记录从reader读了多少
		b.w += n
	}

	// buf的数据copy到p中
	n = copy(p, b.buf[b.r:b.w])
    //记录写入到p中的数量
	b.r += n
	b.lastByte = int(b.buf[b.r-1])
	b.lastRuneSize = -1
	return n, nil
}
```



```

// 读取byte直到分隔符
// 返回切片指向自己的Buf
// bytes不会重复扫描
// If ReadSlice 在找到分隔符前发生error将返回所有在buf中的数据和错误
// ReadSlice 发生 error ErrBufferFull当buf中没有找到分隔符
// Because the data returned from ReadSlice will be overwritten
// by the next I/O operation, most clients should use
// ReadBytes or ReadString instead.
// 当没找到分隔符返回error
func (b *Reader) ReadSlice(delim byte) (line []byte, err error) {
   s := 0 // search start index
   for {
      // 汇编实现 看不明白 猜测是找到分隔符的下标
      if i := bytes.IndexByte(b.buf[b.r+s:b.w], delim); i >= 0 {
         //加上上一轮查找的位置
         i += s
         //截取slice
         line = b.buf[b.r : b.r+i+1]
         //记录buf读取的下标
         b.r += i + 1
         break
      }

      // 如果error返回当前buf中未读的全部返回
      if b.err != nil {
         line = b.buf[b.r:b.w]
         b.r = b.w
         err = b.readErr()
         break
      }

      // Buffer full?
      //当前buf中未能找到对应的分隔符，返回整个buf
      if b.Buffered() >= len(b.buf) {
         b.r = b.w
         line = b.buf
         err = ErrBufferFull
         break
      }

      s = b.w - b.r // do not rescan area we scanned before
	  //填充buf数据再次查找
      b.fill() // buffer is not full
   }

   // Handle last byte, if any.
   if i := len(line) - 1; i >= 0 {
      b.lastByte = int(line[i])
      b.lastRuneSize = -1
   }

   return
}


// 填充buf
func (b *Reader) fill() {
	// Slide existing data to beginning.
	if b.r > 0 {
		copy(b.buf, b.buf[b.r:b.w])
		//将buf中未读的数据copy到buf中首部位置
		//重置r和w 位置
		b.w -= b.r
		b.r = 0
	}

	if b.w >= len(b.buf) {
		panic("bufio: tried to fill full buffer")
	}

	// 最多读maxConsecutiveEmptyReads=100次
	for i := maxConsecutiveEmptyReads; i > 0; i-- {
		//向buf填充新的数据
		n, err := b.rd.Read(b.buf[b.w:])
		if n < 0 {
			panic(errNegativeRead)
		}
		//记录写入buf到的长度
		b.w += n
		if err != nil {
			b.err = err
			return
		}
		if n > 0 {
			return
		}
	}
	b.err = io.ErrNoProgress
}

```









```
// ReadBytes reads until the first occurrence of delim in the input,
// returning a slice containing the data up to and including the delimiter.
// If ReadBytes encounters an error before finding a delimiter,
// it returns the data read before the error and the error itself (often io.EOF).
// ReadBytes returns err != nil if and only if the returned data does not end in
// delim.
// For simple uses, a Scanner may be more convenient.
//过程同ReadString
//和ReadSlice不同的地方ReadSlice如果遇到ErrBufferFull不会继续执行会返回当前的buf和err，ReadSlice会缓存数据继续进行循环
func (b *Reader) ReadBytes(delim byte) ([]byte, error) {
   full, frag, n, err := b.collectFragments(delim)
   // Allocate new buffer to hold the full pieces and the fragment.
   buf := make([]byte, n)
   n = 0
   // Copy full pieces and fragment in.
   for i := range full {
      n += copy(buf[n:], full[i])
   }
   copy(buf[n:], frag)
   return buf, err
}


// collectFragments reads until the first occurrence of delim in the input. It
// returns (slice of full buffers, remaining bytes before delim, total number
// of bytes in the combined first two elements, error).
// The complete result is equal to
// `bytes.Join(append(fullBuffers, finalFragment), nil)`, which has a
// length of `totalLen`. The result is strucured in this way to allow callers
// to minimize allocations and copies.
//如果读到分割符就返回 否则一直读到io.EOF，ErrBufferFull不会影响流程执行，数据会缓存等待找到分隔符或EOF一起返回
func (b *Reader) collectFragments(delim byte) (fullBuffers [][]byte, finalFragment []byte, totalLen int, err error) {
	var frag []byte
	// Use ReadSlice to look for delim, accumulating full buffers.
	for {
		var e error
		frag, e = b.ReadSlice(delim)
		if e == nil { // got final fragment
			break
		}
		//如果不是buf满的error返回
		if e != ErrBufferFull { // unexpected error
			err = e
			break
		}

		// Make a copy of the buffer.
		//保存buf
		buf := make([]byte, len(frag))
		copy(buf, frag)
		fullBuffers = append(fullBuffers, buf)
		//记录长度
		totalLen += len(buf)
	}
	
	totalLen += len(frag)
	return fullBuffers, frag, totalLen, err
}

```

