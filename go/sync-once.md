### sync.Once



```go
//example
func main() {
   once := sync.Once{}
   f := func() {
      fmt.Println(123)
   }
   once.Do(f)
   once.Do(f)
}
//output
1
```

```go
// Once是一个对象只能执行一次
type Once struct {
    //done表明是否被执行
   done uint32
   m    Mutex
}
func (o *Once) Do(f func()) {
    //注意 这里有一个不正确得实现
    //	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
    //Do保证当他返回时，f已经完成
    //这个实现不会保证
    //假如同时两个请求，cas修改成功得请求执行f，失败得则立刻返回，不会等待第一个执行f成功
    //这就是为什么需要doslow，这也为什么修改done要在f执行之后
    //如果未执行 加锁进行二次确认
	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}

```