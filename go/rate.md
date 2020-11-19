### rate

```go
//example
package main

import (
   "golang.org/x/time/rate"
   "log"
   "time"
)

func main() {
	//每秒生成一个token 最大并发2
   r := rate.NewLimiter(1, 2)
   for i := 0; i < 4; i++ {
      go func(i int) {
         log.Println("taskId=", i, " start")
         //是否允许执行
         if r.Allow() {
            log.Println("taskId=", i, " working")
            time.Sleep(time.Second * 5)
            log.Println("taskId=", i, "end working")
         }
      }(i)
   }
   time.Sleep(20e9)
}
```





```go
type Limiter struct {
	mu     sync.Mutex
	limit  Limit     //每秒产生多少token
	burst  int       //token桶容量
	tokens float64   
	last time.Time   //上一次生成token时间
	// lastEvent is the latest time of a rate-limited event (past or future)
	lastEvent time.Time //上一次生成token的时间+等待token生成的时间
}

//r Limit。代表每秒可以向 Token 桶中产生多少 token。Limit是float64
//b表示token桶的容量
func NewLimiter(r Limit, b int) *Limiter {
   return &Limiter{
      limit: r,
      burst: b,
   }
}

// 实际调用 AllowN(time.Now(), 1).
func (lim *Limiter) Allow() bool {
	return lim.AllowN(time.Now(), 1)
}


//如果桶中有N个token返回true，桶中减去n个token
//场景用于丢弃或跳过一些时间
//否则使用Reserve or Wait.
func (lim *Limiter) AllowN(now time.Time, n int) bool {
	return lim.reserveN(now, n, 0).ok
}

// reserveN is a helper method for AllowN, ReserveN, and WaitN.
// maxFutureReserve 表达wait最大等待时间.
// reserveN returns Reservation, not *Reservation, to avoid allocation in AllowN and WaitN.
func (lim *Limiter) reserveN(now time.Time, n int, maxFutureReserve time.Duration) Reservation {
	lim.mu.Lock()
	//允许所有event
	if lim.limit == Inf {
		lim.mu.Unlock()
		return Reservation{
			ok:        true,
			lim:       lim,
			tokens:    n,
			timeToAct: now,
		}
	}
	//返回当前时间 上次token生成时间 当前token数量
	now, last, tokens := lim.advance(now)

	// 计算减去n剩下的token数量
	tokens -= float64(n)

	// Calculate the wait duration
	var waitDuration time.Duration
	if tokens < 0 {
        //如果token负数 计算需要多少时间生成足够的token
		waitDuration = lim.limit.durationFromTokens(-tokens)
	}

	// Decide result
    //如果请求数<=桶的容量&&等待 token的时间<=保留时间
	ok := n <= lim.burst && waitDuration <= maxFutureReserve

	// 生成返回对象
	r := Reservation{
		ok:    ok, //是否允许
		lim:   lim, //*Limiter
		limit: lim.limit,//每秒产生token的数量
	}

	if ok {
		r.tokens = n //请求的token数量
		r.timeToAct = now.Add(waitDuration) //请求时间+等待时间
	}

	// 更新自身状态
	if ok {
		lim.last = now
		lim.tokens = tokens
		lim.lastEvent = r.timeToAct
	} else {
		lim.last = last
	}

	lim.mu.Unlock()
	return r
}
```