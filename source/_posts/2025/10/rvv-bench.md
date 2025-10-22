---
title: rvv-bench测试框架
date: 2025-10-15 11:16:56
tags:
  - RISC-V
  - rvv
  - benchmark
categories:
  - RISC-V
---

上游仓库：

- [camel-cdr/rvv-bench](https://github.com/camel-cdr/rvv-bench) 测试代码
- [camel-cdr/rvv-bench-results](https://github.com/camel-cdr/rvv-bench-results) 测试结果

我fork之后的仓库

- [qiujiandong/rvv-bench](https://github.com/qiujiandong/rvv-bench) 测试代码
- [qiujiandong/rvv-bench-results](https://github.com/qiujiandong/rvv-bench-results)
  测试结果

## 编译

编译之前我调整了一下配置，可以参考这个commit: [qiujiandong/rvv-bench commit:3163e32](https://github.com/qiujiandong/rvv-bench/commit/3163e32b47ba552306cde20722f73ad24a9e2918)
主要是

- 减小了测试数据量的大小，适合在嵌入式平台测试
- 添加了`zvfh`的编译选项,
- 增加了`USER_PERF_EVENT`的宏定义。

然后利用`rvv-bench/bench`目录中的`Makefile`就可以完成编译。

编译完成后得到的是多个二进制文件，可以在Linux上独立测试：

```txt
ascii_to_utf16
ascii_to_utf32
base64_encode
byteswap
chacha20
hist
LUT4
LUT6
mandelbrot
memcpy
memset
mergelines
poly1305
strlen
trans8x8e16
trans8x8e8
utf8_count
```

所有测试程序的输出都是javascript中的一个Object,
将所有测试结果保存到一个`data.js`文件中，写成一个`data`数组，便于后续显示.
类似如下的格式：

```js
// data.js
let data = [
  {
    title: "ascii to utf16",
    labels: [
      "0",
      "scalar",
      "scalar_autovec",
      "rvv_ext_m1",
      "rvv_ext_m2",
      "rvv_ext_m4",
      "rvv_vsseg_m1",
      "rvv_vsseg_m2",
      "rvv_vsseg_m4",
      "rvv_vss_m1",
      "rvv_vss_m2",
      "rvv_vss_m4",
    ],
    ...
  },
...
];
```

## 显示结果

对于结果数据的显示, 我做了一些优化。比如这里有一个`vl128dl128`的文件夹,
如果需要更新测试结果，只需要复制整个文件夹，然后更新`data.js`文件即可。

![results](results.png)

## C代码解析

### 裸机版本

从编译选项中带`-nostdlib`可以看出来，虽然是在linux上跑，但是这个测试代码是不依赖标准库的.
和标准库相关的内容都在`nolibc.h`中实现了，所以如果是需要移植到裸机上跑的话也很方便,
只需要对`nolibc.h`中的内容做裸机的适配即可。

### 自定义测试函数

如果需要测试跟memory大小相关的一些性能，也可以通过这个框架来测试.
比如`memcpy`, 有不同的实现版本，修改`memcpy.c`中的`IMPLS`宏可以控制需要测试的实现版本。

```c
#define IMPLS(f) \
    IFHOSTED(f(libc)) \
    f(musl) \
    f(scalar) \
    f(scalar_autovec) \
    MX(f, rvv) \
    MX(f, rvv_align_dest) \
    MX(f, rvv_align_src) \
    MX(f, rvv_align_dest_hybrid) \
    MX(f, rvv_vlmax) \
    MX(f, rvv_tail) \
    MX(f, rvv_128) \
```

### 测cycle的方式

在Linux上用户一般不能直接读cycle寄存器，但如果Linux配了`CONFIG_PERF_EVENTS=y`,
那么就可以通过`/proc/sys/kernel/perf_user_access`来控制访问cycle寄存器的权限。

从Linux Kernel的[文档](https://docs.kernel.org/admin-guide/sysctl/kernel.html?utm_source=chatgpt.com#perf-user-access-arm64-and-riscv-only)
可以了解到：

```txt
perf_user_access (arm64 and riscv only)
Controls user space access for reading perf event counters.

for arm64 The default value is 0 (access disabled).

When set to 1, user space can read performance monitor counter registers directly.

See Perf for more information.

for riscv When set to 0, user space access is disabled.

The default value is 1, user space can read performance monitor counter registers
through perf, any direct access without perf intervention will trigger an illegal
instruction.

When set to 2, which enables legacy mode (user space has direct access to cycle and
insret CSRs only). Note that this legacy value is deprecated and will be removed
once all user space applications are fixed.

Note that the time CSR is always directly accessible to all modes.
```

代码里有两个宏，`USE_PERF_EVENT` 和 `USE_PERF_EVENT_SLOW`.
其中`USE_PERF_EVENT`就是直接用`rdcycle`指令读cycle,
`USE_PERF_EVENT_SLOW`就是需要通过`perf_event_open`以文件操作的形式获取cycle.
文件操作肯定相对于直接读寄存器更慢一些。

这两种方式不管是哪一种都需要先通过系统调用`perf_event_open`先获取文件描述符,
才能测cycle。所以代码中有一部分绕过glibc库，直接通过`ecall`进行系统调用的做法。

```c
__attribute__((naked))
static int
nolibc_perf_event_open(void *ptr)
{
    __asm__ (
        "li a1, 0\n"
        "li a2, -1\n"
        "li a3, -1\n"
        "li a4, 0\n"
        "li a7, 241\n"
        "ecall\n"
        "ret\n"
    );
}
```

在Linux内核源码中有：

```c
// include/uapi/asm-generic/unistd.h
#define __NR_perf_event_open 241
```

所以需要给a7传入的值是241，也就是`perf_event_open`的系统调用号。
