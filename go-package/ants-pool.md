### ants.Pool

```go
import (
   "fmt"
   "github.com/panjf2000/ants"
   "time"
)

func main() {
	p, _ := ants.NewPool(2)
	var c = make(chan struct{})
	p.Submit(func() {
		fmt.Println(123)
		close(c)
		time.Sleep(5e9)
	})
	fmt.Println(p.Running())
	<-c
}

//output
1
123
```

```go
type Pool struct {
   // 容量
   capacity int32
   // 在运行的数量
   running int32
   // 任务超时事件
   expiryDuration time.Duration
   // 工作队列
   workers []*goWorker
   // release is used to notice the pool to closed itself.
   release int32
   lock sync.Mutex
   // cond for waiting to get a idle worker.
   cond *sync.Cond
   // once makes sure releasing this pool will just be done for one time.
   once sync.Once
   // workerCache speeds up the obtainment of the an usable worker in function:retrieveWorker.
   workerCache sync.Pool
   //panic处理
   panicHandler func(interface{})
    //最大可被阻塞的task数量 默认没有Limit
   maxBlockingTasks int32
   // goroutine already been blocked on pool.Submit
   // protected by pool.lock
   blockingNum int32
   // nonblocking==true Pool.Submit不会被阻塞，如果队列满了返回ErrPoolOverload
    //nonblocking == true MaxBlockingTasks无效
   nonblocking bool
}

// NewPool generates an instance of ants pool.
func NewPool(size int, options ...Option) (*Pool, error) {
	if size <= 0 {
		return nil, ErrInvalidPoolSize
	}
	
	opts := new(Options)
	//处理参数
    for _, option := range options {
		option(opts)
	}
	
	if expiry := opts.ExpiryDuration; expiry < 0 {
		return nil, ErrInvalidPoolExpiry
	} else if expiry == 0 {
		opts.ExpiryDuration = time.Duration(DEFAULT_CLEAN_INTERVAL_TIME) * time.Second
	}

	var p *Pool
    //预先分配workers
	if opts.PreAlloc {
		p = &Pool{
			capacity:         int32(size),
			expiryDuration:   opts.ExpiryDuration,
			workers:          make([]*goWorker, 0, size),
			nonblocking:      opts.Nonblocking,
			maxBlockingTasks: int32(opts.MaxBlockingTasks),
			panicHandler:     opts.PanicHandler,
		}
	} else {
		p = &Pool{
			capacity:         int32(size),
			expiryDuration:   opts.ExpiryDuration,
			nonblocking:      opts.Nonblocking,
			maxBlockingTasks: int32(opts.MaxBlockingTasks),
			panicHandler:     opts.PanicHandler,
		}
	}
	p.cond = sync.NewCond(&p.lock)
    // 定期清理过期worker
	go p.periodicallyPurge()

	return p, nil
}

// Submit submits a task to this pool.
func (p *Pool) Submit(task func()) error {
	if atomic.LoadInt32(&p.release) == CLOSED {
		return ErrPoolClosed
	}
    //查找工作work
	if w := p.retrieveWorker(); w == nil {
		return ErrPoolOverload
	} else {
		w.task <- task
	}
	return nil
}

// 查找可用的work
func (p *Pool) retrieveWorker() *goWorker {
	var w *goWorker
	spawnWorker := func() {
        //如果sync.pool中对象存在
		if cacheWorker := p.workerCache.Get(); cacheWorker != nil {
			w = cacheWorker.(*goWorker)
		} else {
			w = &goWorker{
				pool: p,
                //workerChanCap= p > 1 ? 1 : 0
				task: make(chan func(), workerChanCap),
			}
		}
		w.run()
	}
    
	p.lock.Lock()
	idleWorkers := p.workers
	n := len(idleWorkers) - 1
    //如果有空闲的work
	if n >= 0 {
        //拿最后一个work
		w = idleWorkers[n]
		idleWorkers[n] = nil
		p.workers = idleWorkers[:n]
		p.lock.Unlock()
	} else if p.Running() < p.Cap() {
		//如果运行的数量<最大容量 复用或创建一个work
        p.lock.Unlock()
		spawnWorker()
	} else {
        //如果非阻塞直接返回
		if p.nonblocking {
			p.lock.Unlock()
			return nil
		}
        //循环等待 那道空闲的work
	Reentry:
		if p.maxBlockingTasks != 0 && p.blockingNum >= p.maxBlockingTasks {
			p.lock.Unlock()
			return nil
		}
		p.blockingNum++
        // 阻塞
		p.cond.Wait()
		p.blockingNum--
        // 如果work 都被清理干净了 
		if p.Running() == 0 {
			p.lock.Unlock()
			spawnWorker()
			return w
		}
        
		l := len(p.workers) - 1
		//没懂 锁没释放 为什么会拿不到 work
        if l < 0 {
			goto Reentry
		}
		w = p.workers[l]
		p.workers[l] = nil
		p.workers = p.workers[:l]
		p.lock.Unlock()
	}
	return w
}
//gowork是一个实际的task执行者
// 启动一个goroutine接收task
type goWorker struct {
	pool *Pool
	task chan func()
	// 循环一段事件放入空闲队列
	recycleTime time.Time
}

func (w *goWorker) run() {
    //运行的work+1
	w.pool.incRunning()
	go func() {
		defer func() {
			if p := recover(); p != nil {
                //运行work-1
				w.pool.decRunning()
                //复用独享
				w.pool.workerCache.Put(w)
				if w.pool.panicHandler != nil {
                    //异常处理
					w.pool.panicHandler(p)
				} else {
                    //打印堆栈
					log.Printf("worker exits from a panic: %v\n", p)
					var buf [4096]byte
					n := runtime.Stack(buf[:], false)
					log.Printf("worker exits from panic: %s\n", string(buf[:n]))
				}
			}
		}()
		
		for f := range w.task {
            //pool 关闭
			if f == nil {
				w.pool.decRunning()
				w.pool.workerCache.Put(w)
				return
			}
			f()
            //如果pool没有释放 work放入idle队列 等待任务
			if ok := w.pool.revertWorker(w); !ok {
				break
			}
		}
	}()
}

// Clear expired workers periodically.
func (p *Pool) periodicallyPurge() {
	heartbeat := time.NewTicker(p.expiryDuration)
	defer heartbeat.Stop()

	var expiredWorkers []*goWorker
	for range heartbeat.C {
        //pool关闭
		if atomic.LoadInt32(&p.release) == CLOSED {
			break
		}
		currentTime := time.Now()
		p.lock.Lock()
		idleWorkers := p.workers
		n := len(idleWorkers)
		i := 0
        //如果recycleTime < expiryDuration
		for i < n && currentTime.Sub(idleWorkers[i].recycleTime) > p.expiryDuration {
			i++
		}
		expiredWorkers = append(expiredWorkers[:0], idleWorkers[:i]...)
		if i > 0 {
			m := copy(idleWorkers, idleWorkers[i:])
			for i = m; i < n; i++ {
				idleWorkers[i] = nil
			}
			p.workers = idleWorkers[:m]
		}
		p.lock.Unlock()
        // 通知过期的worker关闭
		for i, w := range expiredWorkers {
			w.task <- nil
			expiredWorkers[i] = nil
		}
		if p.Running() == 0 {
			p.cond.Broadcast()
		}
	}
}
```