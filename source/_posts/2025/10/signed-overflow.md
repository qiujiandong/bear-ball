---
title: 容易被忽略的有符号数溢出问题
date: 2025-10-06 23:41:38
tags:
  - GCC
  - UB
categories:
  - GCC
---

GCC开启`-O2/O3`优化后, 如果不考虑有符号数的溢出, 可能会造成一些无法理解的问题.

## 实例

实际举个例子, 如下面的代码:

```c
// main.c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

int main() {
  for (int i = 0; i < 10; ++i) {
    int32_t a = rand() % 0x10000;
    int16_t b = (int16_t)((a * a) >> 16);
    if (b > 0) {
      printf("b > 0, b = %d\n", b);
    } else {
      printf("b < 0, b = %d\n", b);
    }
  }
}
```

采用不同的优化等级编译并运行:

```shell
❯ gcc -O0 -o main main.c && ./main
b > 0, b = 4816
b > 0, b = 1279
b > 0, b = 23228
b > 0, b = 5248
b < 0, b = -16997
b > 0, b = 8648
b > 0, b = 21989
b > 0, b = 7907
b > 0, b = 970
b > 0, b = 15575
```

```shell
❯ gcc -O2 -o main main.c && ./main
b > 0, b = 4816
b > 0, b = 1279
b > 0, b = 23228
b > 0, b = 5248
b > 0, b = -16997
b > 0, b = 8648
b > 0, b = 21989
b > 0, b = 7907
b > 0, b = 970
b > 0, b = 15575
```

不难发现, `O2`优化的结果中, **有时候`b`打印出来明明是负数, 但判断的时候却是始终认为其是大于0的**,
这就让人感觉非常匪夷所思了.

## 原因分析

`a`是一个[0, 65535)的随机数, `a * a`在`int32_t`类型的表示范围内有可能会溢出,
即超出`int32_t`的表示范围.  
而且编译器在开启`-O2/O3`优化的时候, 会假设不会有有符号数溢出, 因此`a * a`必定大于0.
在这样一个前提下, 下面关于`b`的判断就会始终认为`b`是大于0的.
但实际上在有符号数溢出的情况下, `b`的值是有可能小于0的, 这样一来就导致了上面实际测试中遇到的问题.

## 解决办法

GCC有很多Undefined Behavior(UB), 而且有Undefined Behavior Sanitizer (UBSan) 可以在运行时发现这些隐患.
在编译的时候加上 `-fsanitize=undefined` 选项,
可以参考[GCC文档 -fsanitize=undefined](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html#index-fsanitize_003dundefined),
这样一来运行时就会有如下的错误提示:

```shell
❯ gcc -O2 -fsanitize=undefined -o main main.c && ./main
b > 0, b = 4816
b > 0, b = 1279
b > 0, b = 23228
b > 0, b = 5248
main.c:30:30: runtime error: signed integer overflow: 56401 * 56401 cannot be represented in type 'int'
b < 0, b = -16997
b > 0, b = 8648
b > 0, b = 21989
b > 0, b = 7907
b > 0, b = 970
b > 0, b = 15575
```

## 参考连接

- [UndefinedBehaviorSanitizer (Gentoo wiki)](https://wiki.gentoo.org/wiki/UndefinedBehaviorSanitizer)
- [Improving Application Security with UndefinedBehaviorSanitizer (UBSan) and GCC (Oracle Linux Blog)](https://blogs.oracle.com/linux/post/improving-application-security-with-undefinedbehaviorsanitizer-ubsan-and-gcc)
