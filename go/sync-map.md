### sync.Map

https://zhuanlan.zhihu.com/p/44585993

https://segmentfault.com/a/1190000015242373

<img src="..\images\image-20210330112911073.png" style="zoom:70%;" />

从上图中可以看出，read map 和 dirty map 中含有相同的一部分 `entry`，我们称作是 **normal entries，是双方共享的**，并且满足：其中的 `entry.p` 只会是两种状态:

```
nil      	其中 nil 表示 deleted
expunged 
```

但是 read map 中含有一部分 `entry` **是不属于 dirty map** 的，而这部分 `entry` 就是状态为 `expunged` 状态的 `entry`。

而 dirty map 中有一部分 `entry` 也是**不属于 read map** 的，而这部分其实是来自 `Store` 操作形成的（也就是新增的 `entry`），换句话说就是新增的 `entry` 是出现在 dirty map 中的。



**read map** 是用来进行 lock free 操作的（其实可以读写，但是不能做删除操作，因为一旦做了删除操作，就不是线程安全的了，也就无法 lock free），

**dirty map** 是用来在无法进行 lock free 操作的情况下，需要 lock 来做一些更新工作的对象。



```go
func main() {
   m := sync.Map{}
   
   m.Store("key", "test")
   fmt.Println(m.Load("key"))
   
   m.Delete("key")
   fmt.Println(m.Load("key"))
}
//output
test true
<nil> false
```



```go
https://www.cnblogs.com/buyicoding/p/12117370.html
https://segmentfault.com/a/1190000015242373
// Map像是go的map[interface{}]interface{}但是是并发安全，并发使用不需要额外的锁和条件
// Loads,stores和delete运行在固定时间内

// Map类型是专用的，大多数go code应该使用原生的go map
// 使用锁配合，为了更好的类型安全，更容易维护


// Map针对2种情况做了优化：1）多读少写；2）多协程读、写，覆盖不同的key
type Map struct {
   mu Mutex
   // read包含部分在map中的内容并发安全不需要锁
   // read field是加载安全的，store的时候必须上锁 
   // entriesc存储在read也会会被并发更新在没有上锁的情况下，但是如果更新一个已经被的expunged entry需要entry复制到
   // dirty map并且撤销在持有锁得情况下
   read atomic.Value // readOnly

   //	dirty包含了map中需要加锁才能访问的key
	//	被置为enxugunged的entry不会保存在dirty中
	//	注意点，dirty为nil，read中的amended为false;dirty不为nil，read的amended为true
   dirty map[interface{}]*entry
	
   // misses统计上次read map更新后需要上锁确定key是否存在得次数
    // 一旦足够得misses发生足够覆盖复制dirty map，dirty map将会提升为read map,
    //下次得store 将会生成一个新得dirty copy
   misses int
}

type readOnly struct {
	m       map[interface{}]*entry
	amended bool // true表dirty中包含read不存在的key
}

//	该指针用于标记dirty.map中的一个entry已删除；read中一个entry已删除，则置为nil
var expunged = unsafe.Pointer(new(interface{}))


type entry struct {
	//	p指向存储的interface
	//	p == nil 说明entry已经被删除并且m.dirty == nil
	//	p == expunged 说明entry已经被删除，m.dirty != nil,并且entry不存在于m.dirty
	//  否在enrty在m.read中,存在于m.dirty中如果m.dirty != nil
    //  当m.dirty下一次创建
	p unsafe.Pointer // *interface{}
}

//	为i新建一个entry，其中p保存i的地址
func newEntry(i interface{}) *entry {
	return &entry{p: unsafe.Pointer(&i)}
}


如果从 read map 中能够找到 normal entry 的话，那么就直接 update 这个 entry 就行（lock free)
否则，就上锁，对 dirty map 进行相关操作

上锁之后，需要重新 check 一下 read map 中的内容（这一点是 lockless 里面的一种常见的 pattern），如果发现仍然是 expunged 的，那么会将 expunged 标记为 nil，并且在 dirty map 里面添加相应 key（这里其实就是将这个 entry 从一个 expunged 的 entry 变成了 normal entry）。

func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly)
    //从read中读,如果存在，判断entry是否已经被expunged,如果expunged返回false，否则更新对象
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}
	m.mu.Lock()
    //上锁 二次读
	read, _ = m.read.Load().(readOnly)
    //read中存在 
	if e, ok := read.m[key]; ok {
        //如果entry已经被expunge设置为nil并写入dirty
        // expunge状态只有从reader复制到dirty会产生
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
        //更新
		e.storeLocked(&value)
        //如果在dirty中更新
	} else if e, ok := m.dirty[key]; ok {
		e.storeLocked(&value)
	} else {
        //如果dirty中也没有 说明dirty未重建或新建
		if !read.amended {
			//如果dirty为空 初始化，并从read未expunge的key复制到dirty	
			m.dirtyLocked()
            //更新m.read.amended dirty中存在read中没有的key
			m.read.Store(readOnly{m: read.m, amended: true})
		}
        //新增key
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}

func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}
	
	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
	for k, e := range read.m {
        //把read中未被expunge的移动到dirty中
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
        // 把删除状态变为expunged
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}



func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
    // read中不存在 并且dirty已经使用
	if !ok && read.amended {
		m.mu.Lock()
		//二次读
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
            // dirty中找
			e, ok = m.dirty[key]
			//read中未命中
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

func (m *Map) missLocked() {
	m.misses++
    //日过未命中次数小于当前dirty
	if m.misses < len(m.dirty) {
		return
	}
    //dirty提升为read
	m.read.Store(readOnly{m: m.dirty})
	//更新属性
    m.dirty = nil
	m.misses = 0
}

// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
	m.LoadAndDelete(key)
}

// LoadAndDelete删除一个key，如果存在key返回对应的value
//通load差不多 主要流程
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
    //如果在read中不存在并且存在dirty map
	if !ok && read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
            delete(m.dirty, key)
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
        // 状态变成nil
		return e.delete()
	}
	return nil, false
}

func (e *entry) delete() (value interface{}, ok bool) {
	for {
		p := atomic.LoadPointer(&e.p)
       
		if p == nil || p == expunged {
			return nil, false
		}
        //状态变更为nil 返回p的值
        //store的时候会使用这个值
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return *(*interface{})(p), true
		}
	}
}

// LoadOrStore返回存在的key当前value
// 否则存储并返回value
// loaded返回true表明存在，false表明不存在
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
	// Avoid locking if it's a clean hit.
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		actual, loaded, ok := e.tryLoadOrStore(value)
		if ok {
			return actual, loaded
		}
	}

	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
    //双重检查
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		actual, loaded, _ = e.tryLoadOrStore(value)
	} else if e, ok := m.dirty[key]; ok {
		actual, loaded, _ = e.tryLoadOrStore(value)
		m.missLocked()
	} else {
		if !read.amended {
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
		actual, loaded = value, false
	}
	m.mu.Unlock()

	return actual, loaded
}


// tryLoadOrStore atomically loads or stores a value if the entry is not
// expunged.
//
// If the entry is expunged, tryLoadOrStore leaves the entry unchanged and
// returns with ok==false.
func (e *entry) tryLoadOrStore(i interface{}) (actual interface{}, loaded, ok bool) {
	p := atomic.LoadPointer(&e.p)
	//已经删除 返回
    if p == expunged {
		return nil, false, false
	}
    //存在返回
	if p != nil {
		return *(*interface{})(p), true, true
	}

	// Copy the interface after the first load to make this method more amenable
	// to escape analysis: if we hit the "load" path or the entry is expunged, we
	// shouldn't bother heap-allocating.
	ic := i
	for {
        //cas赋值
		if atomic.CompareAndSwapPointer(&e.p, nil, unsafe.Pointer(&ic)) {
			return i, false, true
		}
		p = atomic.LoadPointer(&e.p)
		if p == expunged {
			return nil, false, false
		}
		if p != nil {
			return *(*interface{})(p), true, true
		}
	}
}
```