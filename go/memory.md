### 内存分配



https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/

https://louyuting.blog.csdn.net/article/details/102945046



### 分配方法 

编程语言的内存分配器一般包含两种分配方法，一种**线性分配器**（Sequential Allocator，Bump Allocator），另一种是**空闲链表分配器**（Free-List Allocator），这两种分配方法有着不同的实现机制和特性，本节会依次介绍它们的分配过程。



线性分配（Bump Allocator）是一种高效的内存分配方法，但是有较大的局限性。当我们使用线性分配器时，只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置，即移动下图中的指针：

![https://img.draveness.me/2020-02-29-15829868066435-bump-allocator.png]()





