### os



```go
//example
package main

import (
   "log"
   "os"
)

func main() {
   f, err := os.Open("D:\\gocode\\test\\io\\readall.go")
   defer func() {
      if err != nil {
         log.Println(err)
      }
   }()
   if err != nil {
      return
   }
   buf := make([]byte, 4096)
   n, err := f.Read(buf)
   if err != nil {
      return
   }
   log.Println(string(buf[:n]))
}
```





```go
//windows下
//如果打开文件成功返回句柄 否则返回*PathError.
func Open(name string) (*File, error) {
   return OpenFile(name, O_RDONLY, 0)
}

//打开文件
//如果文件不存在，并且flag O_CREATE,将会创建一个新文件
// openFileNolog is the Windows implementation of OpenFile.
func OpenFile(name string, flag int, perm FileMode) (*File, error) {
	testlog.Open(name)
	f, err := openFileNolog(name, flag, perm)
	if err != nil {
		return nil, err
	}
    //文件追加模式
	f.appendMode = flag&O_APPEND != 0

	return f, nil
}

// openFileNolog is the Windows implementation of OpenFile.
func openFileNolog(name string, flag int, perm FileMode) (*File, error) {
	if name == "" {
		return nil, &PathError{"open", name, syscall.ENOENT}
	}
    //打开文件成功返回 系统调用
	r, errf := openFile(name, flag, perm)
	if errf == nil {
		return r, nil
	}
    //当作文件夹处理
	r, errd := openDir(name)
	if errd == nil {
        //如果flag O_WRONLY或者O_RDWR 返回文件夹写错误
		if flag&O_WRONLY != 0 || flag&O_RDWR != 0 {
			r.Close()
			return nil, &PathError{"open", name, syscall.EISDIR}
		}
		return r, nil
	}
	return nil, &PathError{"open", name, errf}
}

```