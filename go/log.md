### log

```go
//example
package main

import (
   "log"
   "os"
)

func main() {
   l := log.New(os.Stdout, "my_log: ", log.LstdFlags)
   l.Println("first")
   l.Printf("%s", "second")
}

-----------------output-------------
my_log: 2020/11/23 16:20:10 first
my_log: 2020/11/23 16:20:10 second
```

```go

//默认对象
var std = New(os.Stderr, "", LstdFlags)
// Println calls Output to print to the standard logger.
// Arguments are handled in the manner of fmt.Println.
func Println(v ...interface{}) {
	std.Output(2, fmt.Sprintln(v...))
}

//New创一个新的Logger out指示日志要 出书到什么地方
//prefix出现在每条日志的开始，如果Lmsgprefix出现在每条日志尾部
//flag定义了logging的特性
func New(out io.Writer, prefix string, flag int) *Logger {
   return &Logger{out: out, prefix: prefix, flag: flag}
}


//Logger代表了一个活跃的logging对象，每条日志调用Writer's Write method
//Logger可以同事被多个goroutine使用，保证了写入Writer的顺序
type Logger struct {
	mu     sync.Mutex // 锁保证并发
	prefix string     // prefix on each line to identify the logger (but see Lmsgprefix)
	flag   int        // properties
	out    io.Writer  // destination for output
	buf    []byte     // for accumulating text to write
}


// Println 调用 l.Output打印日志
//参数的处理同fmt.Println
func (l *Logger) Println(v ...interface{}) { l.Output(2, fmt.Sprintln(v...)) }

// Printf 调用 l.Output打印日志
//参数的处理同fmt.Printf
func (l *Logger) Printf(format string, v ...interface{}) {
	l.Output(2, fmt.Sprintf(format, v...))
}


//Output写入日志的输出,string s包含要在prefix之后打印的内容
//如果上一次s的结尾是不是换行将被插入一个新行
//Calldepth用户恢复oc尽管大多数情况下预定是2
func (l *Logger) Output(calldepth int, s string) error {
	now := time.Now() // get this early.
	var file string
	var line int
	l.mu.Lock()
	defer l.mu.Unlock()
    //
	if l.flag&(Lshortfile|Llongfile) != 0 {
        //释放锁 caller调用比较费时
        //是否此时会发生抢占
		l.mu.Unlock()
		var ok bool
        //调用信息
		_, file, line, ok = runtime.Caller(calldepth)
		if !ok {
			file = "???"
			line = 0
		}
        //上锁
		l.mu.Lock()
	}
    //清空缓存
	l.buf = l.buf[:0]
    //拼装头部
	l.formatHeader(&l.buf, now, file, line)
    //写入参数
	l.buf = append(l.buf, s...)
    //检查最后一位是不是换行
	if len(s) == 0 || s[len(s)-1] != '\n' {
		l.buf = append(l.buf, '\n')
	}
    //写入输出对象
	_, err := l.out.Write(l.buf)
	return err
}
```