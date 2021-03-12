### 堆算法

https://www.cnblogs.com/hello-shf/p/11393655.html

堆是一个完全二叉树（其实就是对于一棵树，每一层从左往右编号，不能中断编号）

父节点 = (子节点-1)/2

左子节点 = 父节点*2+1

右子节点 = 父节点*2+2

堆要求**孩子节点要小于等于父亲节点**（如果是最小堆则大于等于其父亲节点）



```go

//example
package main

import (
   "container/heap"
   "fmt"
)

type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
   *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
   old := *h
   n := len(old)
   x := old[n-1]
   *h = old[0 : n-1]
   return x
}

func main() {
	var v = &IntHeap{5, 7, 2, 4, 1}
	heap.Init(v)
	heap.Push(v, 6)
	for v.Len() > 0 {
		fmt.Println(heap.Pop(v))
	}
}
//output
1
2
4
5
6
7
```



```go
//heap.Interface
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}

//sort.Interface
type Interface interface {
	//返回切片长度
	Len() int
    //Less返回下标i的元素应该比下标j的元素小
	Less(i, j int) bool
	// 交换i,j下标元素
	Swap(i, j int)
}

//创建堆
func Init(h Interface) {
   //元素长度
   n := h.Len()
   for i := n/2 - 1; i >= 0; i-- {
       //下沉元素
       //i需要调整的最后叶子节点的父节点
      down(h, i, n)
   }
}

func down(h Interface, i0, n int) bool {
	i := i0
	for {
        //左儿子
		j1 := 2*i + 1
        //判断左儿子是否存在
		if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
			break
		}
		j := j1 // left child
        //j2右儿子
        //左右两个儿子找 找出最小的
		if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
			j = j2 // = 2*i + 2  // right child
		}
        //父节点和左右儿子中较小的比较
		if !h.Less(j, i) {
			break
		}
        //如果父节点比儿子节点大 交换
		h.Swap(i, j)
        //父节点下沉 再次比对
		i = j
	}
	return i > i0
}


// Push pushes the element x onto the heap.
// The complexity is O(log n) where n = h.Len().
func Push(h Interface, x interface{}) {
	//切片append元素
    h.Push(x)
    //提升最后一个元素
	up(h, h.Len()-1)
}


func up(h Interface, j int) {
	for {
        //找到父节点
		i := (j - 1) / 2 
        //父节点 不需要swap
		if i == j || !h.Less(j, i) {
			break
		}
        //交换
		h.Swap(i, j)
		//当前父节点的父节点
        j = i
	}
}

```