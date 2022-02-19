### 基础



```
#include <stdio.h>                //涵盖输出输入方法

#include "ctype.h"								//涵盖ascii标准函数
isalnum()
isalpha()
isblack()
...



c99支持
_Bool
#include <stdbool.h> 可以使用bool类型 值true｜false
数组定义  int arr[6] = {[5] = 100}
int n = 10;int arr[n];

// 数组在函数调用的时候是个指针，代表数组的起始元素
int arr[6];
// int *arr 定义等于 int arr[]
int sum(int * arr, int l);

变长数组
int m = 5;int n = 6; int arr[m][n];	//函数内部定义，声明时不能初始化，创建后不能修改数组大小。




i++ ++i 注意

表达式顺序不能确定， 逻辑表达式从左到右
```

