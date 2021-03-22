### LRU

https://blog.csdn.net/csdnsevenn/article/details/90152621



```go
type data struct {
   key, val int
}
type LRUCache struct {
   cache map[int]*list.Element
   *list.List
   capacity int
}

func Constructor(capacity int) LRUCache {
   return LRUCache{
      cache:    map[int]*list.Element{},
      List:     list.New(),
      capacity: capacity,
   }
}

func (this *LRUCache) Get(key int) int {
   val, ok := this.cache[key]
   if !ok {
      return -1
   }
   this.List.MoveToFront(val)
   return val.Value.(data).val
}

func (this *LRUCache) Put(key int, value int) {
   e, ok := this.cache[key]
   if !ok {
      e = this.List.PushFront(data{key, value})
      this.cache[key] = e
   } else {
      e.Value = data{key, value}
   }
   this.List.MoveToFront(e)
   if len(this.cache) > this.capacity {
      rs := this.List.Back()
      this.List.Remove(rs)
      delete(this.cache, rs.Value.(data).key)
   }
}
```