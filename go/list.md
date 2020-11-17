### list



```go
//example
package main

import (
   "container/list"
   "fmt"
)

func main() {
	//初始化
   l := list.New()
   s := []string{"a", "b", "c", "d", "e", "f"}
   //添加到链表尾部
    for _, val := range s {
      l.PushBack(val)
   }
    //插入头部
   l.PushFront("head")
    //获取链表尾部
	fmt.Println(l.Back().Value)
	fmt.Println(l.Len())
    
    
    //从头部移除元素
    for {
		e := l.Front()
		if e == nil {
			break
		}
		v := l.Remove(e)
		fmt.Println(v)
	}
    
}
```





```go

type List struct {
	root Element // 头元素
	len  int     // 链表长度
}

// 链表元素
type Element struct {
	//双向链表，指向上一个，下一个元素
	//简化实现，内部实现list做位一个环，&l.root最后一个元素的下一个元素
	next, prev *Element
	// 元素属于哪一个链表
	list *List
	// 链表保存的值
	Value interface{}
}

// New returns an initialized list.
func New() *List { 
    return new(List).Init() 
}

// 初始化头元素
func (l *List) Init() *List {
    //初始化的时候，指向自己
    l.root.next = &l.root
	l.root.prev = &l.root
	l.len = 0
	return l
}

//再list尾部插入element
func (l *List) PushBack(v interface{}) *Element {
    //如果root.next == nil初始化
	l.lazyInit()
   	//通过root.prev插入新元素
    //prev指向的是链表的尾部
	return l.insertValue(v, l.root.prev)
}

// insertValue is a convenience wrapper for insert(&Element{Value: v}, at).
func (l *List) insertValue(v interface{}, at *Element) *Element {
	return l.insert(&Element{Value: v}, at)
}

//e待添加的元素，at添加元素的上一个位置
func (l *List) insert(e, at *Element) *Element {
	//元素prev指向上一个元素
    e.prev = at
    //元素的next指向上一个元素的next
    //如果末尾插入那就是指向头 否则就是下个元素
	e.next = at.next
    //更新上一个元素的next为自己
    //at.next = e
	e.prev.next = e
	//更新下一个元素上一个元素为自己
    e.next.prev = e
    //更新元素所属
	e.list = l
	l.len++
	return e
}

```





```go
//example



//从列表移除元素
//返回元素的值
//元素不能为nil
func (l *List) Remove(e *Element) interface{} {
   if e.list == l {
      // if e.list == l, l must have been initialized when e was inserted
      // in l or l == nil (e is a zero Element) and l.remove will crash
      l.remove(e)
   }
   return e.Value
}

//从list移除e元素
func (l *List) remove(e *Element) *Element {
    //元素的前一个元素的next指向e的next
	e.prev.next = e.next
    //元素的下一个元素的prev指向e.prev
	e.next.prev = e.prev
    //
	e.next = nil // avoid memory leaks
	e.prev = nil // avoid memory leaks
	e.list = nil
	l.len--
	return e
}

```