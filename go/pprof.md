### pprof



```
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof"
	"time"
)

func main() {
	// 是否阻塞的开关
	var flag bool
	testCh := make(chan int)

	go func() {
		// pprof 监听的端口 8080
		if err := http.ListenAndServe("0.0.0.0:8080", nil); err != nil {
			fmt.Printf("%v", err)
		}
	}()

	go func() {
		// 永远为 false 的条件
		if flag {
			<-testCh
		}

		// 死循环
		select {}
	}()

	// 每秒执行100次
	tick := time.Tick(time.Second / 100)
	for range tick {
		ch1(testCh)
	}
}

func ch1(ch chan<- int) {
	go func() {
		defer fmt.Println("ch1 stop")
		// 给 ch channel 写入数据
		ch <- 0
	}()
}

```

http://127.0.0.1:8080/debug/pprof/



brew install graphviz



默认存储到 `$home/pprof` 目录中（如果设置了 `PPROF_TMPDIR` 这个环境变量，则需要到这个环境变量设置的地址中查找 profile 文件）



go tool pprof  http://localhost:8080/debug/pprof/goroutine

go tool pprof -http=:8080 [profiling file path]



命令

- top：默认展示前 10 条样本计数最高的，后面接数字，则显示指定数字的条目，如：top3
- traces：输出所有 profile 的统计信息
- list：输出给定正则表达式匹配的方法源码，list 后面是可以接正则表达式
- tree：输出所有调用关系
- web：生成 profile 的 svg 矢量图片并用 web 打开，如果不指定参数则显示所有，给定指定方法，则显示指定的方法
- pdf：生成 pdf 的 profile 文件，里面展示 profile 图片的内容
- cum：按照累计的 profile 样本量排序
- flat：按照当前函数占用的 profile 排序







```
// 需要引入的包
import _ "runtime/pprof"
 
 
// file 是 os.File 结构，用来创建 pprof 要写入的文件
// 开始 trace
trace.Start(file)
// 结束 trace，并将trace信息完全写入到文件中
defer trace.Stop()


package main
import (	
    "os"
    "runtime/trace"
)

func main() {
    f, err := os.Create("trace.out")	
    if err != nil {		
       panic(err)
    }	
    defer f.Close()

    err = trace.Start(f)
     if err != nil {
 	panic(err)
    }	
    defer trace.Stop()  
    // Your program here
}





curl "http://localhost:8080/debug/pprof/trace?seconds=50" > trace.log


go tool trace [trace file path]
```

