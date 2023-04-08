---
title: "typedef with array"
linkTitle: "typedef_array"
date: 2023-04-08
description: >
 .
---

# Precaution of "typedef an Array"

## Issues

Usually we use 'typedef' to define a new data type.Sometimes we use 'typedef' to define a array as well. The following shows an example to define an array as a new data type. We use 'typedef' to define a new data type MacAddress which consists of 6 bytes.

```C
typedef unsigned char MacAddress[6];
```

Let's check if the following code works correctly or not.

```C
typedef unsigned char MacAddress[6];
MacAddress macAddr, inMacAddr;

...
memcpy(&macAddr, &inMacAddr, 6);
```

Answers: It depends.
Why?
Let's check the following first:

```C
#include "stdio.h"
#include "stdlib.h"

typedef unsigned char MacAddress[6];

int main()
{
   MacAddress macAddr;
   printf("Output:\n %p\n %p\n %p\n", macAddr, &macAddr, &macAddr[0]);
   return 0;
}
```

这个大家都比较熟悉了。MacAddress是一个数组类型，所以其变量名macAddr就是其首地址。@macAddr是取这个变量的地址，当然也就是macAddr的首地址了。@macAddr[0]是取数组第一个元素的地址，这个地址等于整个数组的首地址。所以这三个指针是相等的，都等于整个数组的首地址。如下所示：

```shell
Output:
  0x7fff39126150
  0x7fff39126150
  0x7fff39126150
```

![Example image](/imgs/macAddr-1.png)

以上是我们在使用数组时的常见理解。
好的，我们再看下面一段代码：

```C
#include "stdio.h"
#include "stdlib.h"

typedef unsigned char MacAddress[6];

void show(MacAddress param_macAddr)
{
   printf("Output:\n  %p\n  %p\n", param_macAddr, &param_macAddr);
}

int main()
{
   MacAddress macAddr;
   show(macAddr);
   return 0;
}

void show(unsigned char *param_macAddr)
{
   printf("Output:\n  %p\n  %p\n", param_macAddr, &param_macAddr);
}
```

这里param_macAddr是一个新的临时创建的指针变量，这个指针变量指向原来的macAddr；但这时的@param_macAddr已经指向这个新的临时变量了。所以在上例中的输出，两个指针是不一样的。

![Example image](/imgs/macAddr-2.png)

当使用MacAddress这样的数据类型做函数参数时，形式上像是值传递，直接使用变量名来传递参数；效果上有点像引用传递，在函数中修改变量内容，原调用者的数据也变化了；实质上却是指针传递，只是创建了数组对象的指针作为参数传给函数。

这个问题是由数组数据的特性引起的，本身和typedef没有关系。只是使用typedef定义数组数据类型后，新数据类型看上去像一个一般数据类型，但使用时，却仍然是个指针数据类型。

这就解释了最初的那个memcpy的那个例子。如果memcpy的两个参数中有一个是函数参数，其运行结果都不是我们期望的。

## 如何避免误用.

不再对数组数据取地址. 既然macAddr就是期望的数组首地址，就不需要再加取地址符@macAddr。我们已经了解在不同的情况下，@macAddr有二义性，为了减少代码的二义性，我们也不该再使用它。

不使用typedef定义数组数据结构 使用数据结构或者类来定义新的数据类型。数据结构或类有更好的封装性和扩展性。

使用参数引用做为函数参数 如果使用参数引用来传递数组数据，可以避免上述问题。如下代码所示。不过，对解决数组做函数参数的问题，前面两个方法更直接，更易懂。

```C
#include "stdio.h"
#include "stdlib.h"

typedef unsigned char MacAddress[6];

void show(MacAddress& param_macAddr)
{
   printf("%p-%p\n", param_macAddr, &param_macAddr);
}

int main()
{
   MacAddress macAddr;
   printf("%p-%p-%p\n", macAddr, &macAddr, &macAddr[0]);
   show(macAddr);
   return 0;
}

Output: 0x7fff39126150-0x7fff39126150-0x7fff39126150
Output: 0x7fff39126150-0x7fff39126150
```