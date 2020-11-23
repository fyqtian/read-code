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
```