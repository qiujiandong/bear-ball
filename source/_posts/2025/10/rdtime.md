---
title: RISC-V中利用rdtime估计cycle数
date: 2025-10-10 00:17:24
tags:
  - rdtime
  - RISC-V
categories:
  - RISC-V
---

在RISC-V Linux中, `rdtime`或者`rdcycle`可以用来获取和时间相关的统计量。

- `rdtime`采用的是较低精度的时钟源, 但是这个时钟是以恒定频率工作的，所以可以用来稳定计时。
- `rdcycle`相对地，CPU的cycle所对应的时钟可能是动态变化的，但它也是用来精确衡量算力消耗的单位。

假定CPU以恒定频率工作，那么`rdtime`和`rdcycle`就是呈固定的线性关系，因此可以由`rdtime`的结果来推算cycle数。

## 测试代码

```c
// main.c
#include <stdio.h>

extern unsigned long test();

int main() { printf("cycle: %lu\n", test()); }
```

```asm
# test.S
.global test

test:
    li      t0, 10000
    rdtime  t1
1:
    ld      a0, 0(sp)
    ld      a1, 8(sp)
    ld      a2, 16(sp)
    ld      a3, 24(sp)
    ld      a4, 32(sp)
    ld      a5, 40(sp)
    ld      a6, 48(sp)
    ld      a7, 56(sp)
    ld      s0, 64(sp)
    ld      s1, 72(sp)
    ld      s2, 80(sp)
    ld      s3, 88(sp)
    ld      s4, 96(sp)
    ld      s5, 104(sp)
    ld      s6, 112(sp)
    ld      s7, 120(sp)
    addi    t0, t0, -1
    bnez    t0, 1b
    rdtime  t0
    sub     a0, t0, t1
    ret

```

## 编译

```shell
riscv64-unknown-linux-gnu-gcc -march=rv64imafdc -mabi=lp64d -static -o main main.c test.S
```

## 测试

```shell
./main
cycle: 2706
```

经过`rdtime`统计得到的结果是2706个计数值，那么`rdtime`的参考频率是多少？  
可以通过`timebase-frequency`的值来确定。

## 结果分析

```shell
> xxd /proc/device-tree/cpus/timebase-frequency
00000000: 019b fcc0                                ....
```

`timebase-frequency`是以大端形式存放的，实际的值应该是 `27000000`

![timebase](timebase-frequency.png)

如果CPU是固定以1.6GHz工作的，那么就可以计算出实际的cycle数：

```shell
>>> 2706/27000000*1600000000
160355.55555555556
```

从测试的结果可以看出，代码中大约执行160000次`ld`指令，推算得出大约耗时160355 cycle，结果也是比较合理的。
