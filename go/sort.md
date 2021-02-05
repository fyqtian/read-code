### sort

#### sort.SearchInts

```go
func main() {
	fmt.Println(sort.SearchInts([]int{1, 2, 3, 5}, 4))
}
// 一般情况下得方便包装
// SearchInts在排序得slice中搜索x并返回下标
// 如果x不存在返回得是x需要插入得位置
// slice必须被升序排序
func SearchInts(a []int, x int) int {
	return Search(len(a), func(i int) bool { return a[i] >= x })
}

func Search(n int, f func(int) bool) int {
	// Define f(-1) == false and f(n) == true.
	// Invariant: f(i-1) == false, f(j) == true.
	i, j := 0, n
    //如果左边界<右边界
	for i < j {
        // 中位数下标
		h := int(uint(i+j) >> 1) // avoid overflow when computing h
		// i ≤ h < j
        // 如果中位数<X 移动左边界
		if !f(h) {
			i = h + 1 // preserves f(i-1) == false
		} else {
			j = h // preserves f(j) == true
		}
	}
	// i == j, f(i-1) == false, and f(j) (= f(i)) == true  =>  answer is i.
	return i
}

```