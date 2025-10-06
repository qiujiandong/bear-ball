---
title: pragma pack()在GCC编译时不接受宏参数
date: 2025-10-06 18:20:33
tags:
  - GCC
  - MSVC
categories:
  - GCC
---

`#pragma pack()` 可以用来让结构体内的数据紧凑地组织在一起。

最近遇到一些在MSVC下适用的代码，换成GCC编译器之后就不正常的情况。

## 原因分析

经过查找一些资料和尝试，发现主要原因是使用GCC编译器时， `#pragma pack()` 中指定的对齐字节数,
不能通过宏来传递，而MSVC则可以通过宏来传递。

## 测试代码

定义了一个结构体`A`，在使用`#pragma pack(2)`时, `A`占6字节，如果不使用则占8字节。

```c
// main.c
#include <stdint.h>
#include <stdio.h>

#if USE_MACRO
#define ALIGN 2
#pragma pack(ALIGN)
#else
#pragma pack(2)
#endif
typedef struct {
    uint16_t a;
    uint32_t b;
} A;
#pragma pack()

int main() {
    printf("sizeof A: %u\n", sizeof(A));
    return 0;
}
```

## 测试结果

```shell
❯ gcc -o main -DUSE_MACRO=0 main.c && ./main
sizeof A: 6
❯ gcc -o main -DUSE_MACRO=1 main.c && ./main
main.c:7:14: warning: unknown action ‘ALIGN’ for ‘#pragma pack’ - ignored [-Wpragmas]
    7 | #pragma pack(ALIGN)
      |              ^~~~~
sizeof A: 8
```

测试后可以发现在不使用宏定义的情况下，结果是正确的,
但是如果使用宏定义，GCC编译时也会有相应的warning提示。

另外，如果在windows上采用MSVC编译器，即使使用宏定义，结果也会是正确的。
