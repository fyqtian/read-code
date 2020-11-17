### ring

```go
//example
package main

import (
   "container/ring"
   "fmt"
   "time"
)

func main() {
   r := ring.New(4)
   r.Value = 100
   r = r.Next()
   r.Value = 200

   for r != nil {
      fmt.Println(r.Value)
      r = r.Next()
      time.Sleep(1e9)
   }

   fmt.Println(r.Len())
}
```



```
type Ring struct {
	next, prev *Ring
	Value      interface{} // for use by client; untouched by this library
}
// 创建一个n个元素的环
func New(n int) *Ring {
   if n <= 0 {
      return nil
   }
   //头节点
   r := new(Ring)
   p := r
   for i := 1; i < n; i++ {
      p.next = &Ring{prev: p}
      p = p.next
   }
   p.next = r
   //指向尾部
   r.prev = p
   return r
}


//遍历ring一圈调用f
func (r *Ring) Do(f func(interface{})) {
	if r != nil {
		f(r.Value)
		for p := r.Next(); p != r; p = p.next {
			f(p.Value)
		}
	}
}

//Link
//如果r和s是指向同个ring 将会移除除了当前当前r和s之前的差值
//如果r和s指向不同的的ring 将会创建一个新的ring，s的值插入到r之后
func (r *Ring) Link(s *Ring) *Ring {
	n := r.Next()
	if s != nil {
		p := s.Prev()
		// Note: Cannot use multiple assignment because
		// evaluation order of LHS is not specified.
		
		r.next = s
		s.prev = r
		//n之后接到p之后
		//n的next还是指向a
		n.prev = p
		//p之后接n
		//p还是接在s之后
		p.next = n
	}
	return n
}


//删除n&r.Len()个元素
func (r *Ring) Unlink(n int) *Ring {
	if n <= 0 {
		return nil
	}
	return r.Link(r.Move(n + 1))
}

//移动n%r.Len()个位置
//负数往左移动
//正数往右移
//example n = 1
//0 1 2 3
//1 2 3 0
func (r *Ring) Move(n int) *Ring {
	if r.next == nil {
		return r.init()
	}
	switch {
	case n < 0:
		for ; n < 0; n++ {
			r = r.prev
		}
	case n > 0:
		for ; n > 0; n-- {
			r = r.next
		}
	}
	return r
}

```

