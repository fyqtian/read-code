### Context

每一个 context.Context 都会从最顶层的 Goroutine 一层一层传递到最下层。context.Context 可以在上层 Goroutine 执行出现错误时，将信号及时同步给下层



```go
//example
package main

import (
   "context"
   "log"
   "time"
)

func main() {
   ctx := context.Background()

   ctx, _ = context.WithTimeout(ctx, 2*time.Second)
   log.Println("start")

   go subMission(ctx, 1)
   go subMission(ctx, 2)

   select {
   case <-ctx.Done():
      log.Println(ctx.Err())
   }
   
   time.Sleep(5e9)
}

func subMission(ctx context.Context, taskId int) {
   log.Printf("taskId=%d  start work", taskId)
   for {
      select {
      case <-ctx.Done():
         log.Printf("taskId=%d end work", taskId)
         return
      default:
      }
      log.Printf("taskId=%d  working", taskId)
      time.Sleep(time.Second)
   }
}
//output
2020/12/29 17:43:24 start
2020/12/29 17:43:24 taskId=2  start work
2020/12/29 17:43:24 taskId=1  start work
2020/12/29 17:43:24 taskId=1  working
2020/12/29 17:43:24 taskId=2  working
2020/12/29 17:43:25 taskId=2  working
2020/12/29 17:43:25 taskId=1  working
2020/12/29 17:43:26 context deadline exceeded
2020/12/29 17:43:26 taskId=2 end work
2020/12/29 17:43:26 taskId=1 end work
```





```go
// Context's methods may be called by multiple goroutines simultaneously.
type Context interface {
    //返回被取消的事件，ok==true deadline已经被set
   Deadline() (deadline time.Time, ok bool)

   //返回一个 Channel，这个 Channel 会在当前工作完成或者上下文被取消之后关闭，多次调用 Done 方法会返回同一个 Channel

   Done() <-chan struct{}

   //如果超时 返回DeadlineExceeded
    //如果cancel 返回Canceled的错误
   Err() error

   // Value — 从 context.Context 中获取键对应的值，对于同一个上下文来说，多次调用 Value 并传入相同的 Key 会返回相同的结果，该方法可以用来传递请求特定的数据；
   Value(key interface{}) interface{}
}
```





```go
//Backgrouod顶层父节点
var (
   background = new(emptyCtx)
   todo       = new(emptyCtx)
)

func Background() Context {
   return background
}

func TODO() Context {
   return todo
}

// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```





```go
//
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
   if parent == nil {
      panic("cannot create context from nil parent")
   }
   //包装一个新的子context
   c := newCancelCtx(parent)
   //同步父节点和子节点
   propagateCancel(parent, &c)
   return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context    					   //父context

	mu       sync.Mutex            // protects following fields
    done     chan struct{}         // 延迟创建 调用ctx.Done()创建
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}

// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // 父context是不能取消类型得 返回 比如emptyCtx valueCtx
	}

	select {
	case <-done:
		//如果父已经关闭，子节点取消
        //关闭done，取消子子节点
		child.cancel(false, parent.Err())
		return
	default:
	}
	//判断父节点是不是cancelCtx类型
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
            //认为父已经取消 子节点取消
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
            //子节点加入
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
        //for test
		atomic.AddInt32(&goroutines, +1)
        //过期类型得 启goroutine监听
		go func() {
			select {
             //监听父节点取消
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}


//关闭c.done 关闭children
//removeFromParent=true 父节点上移除context
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
        //父类型如果是cancel移除掉当前context
		removeChild(c.Context, c)
	}
}
```





```go


func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
   return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
    //
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// 如果父节点比当前早结束 转换成cancelCtx
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
    //绑定关系
	propagateCancel(parent, c)
	dur := time.Until(d)
    //如果已经超时 取消
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
        //如果还未超市 设置定时器
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}

// A timerCtx carries a timer and a deadline. It embeds a cancelCtx to
// implement Done and Err. It implements cancel by stopping its timer then
// delegating to cancelCtx.cancel.
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()  
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}


```