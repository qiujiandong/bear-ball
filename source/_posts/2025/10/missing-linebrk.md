---
title: 消失的换行符
date: 2025-10-16 22:35:28
tags:
  - GCC
categories:
  - GCC
---

今天遇到一个问题, 在`gcc`编译的时候, 常量字符串末尾的换行符`\n`没有出现在反汇编代码中.

---

代码非常简单:

```c
#include <stdio.h>
int main() {
  printf("hello\n");
  return 0;
}
```

```bash
objdump -s -j .rodata main

main:     file format elf64-x86-64

Contents of section .rodata:
 2000 01000200 68656c6c 6f00               ....hello.
```

从二进制的数据来看, `68`, `65`, `6c`, `6c`, `6f`对应的就是`hello`,
但是随后就是`00`表示字符串结尾了, `\n`对应的ASCII码是`0a`却没有出现.

后来通过反汇编的代码发现原因很简单, 就是`puts`函数能够自带换行符地打印输出,
在打印带换行符的常量字符串的时候, 直接用`puts`函数替换了`printf`,
从而节省了一个字符.

```asm
0000000000001149 <main>:
    1149:	f3 0f 1e fa          	endbr64
    114d:	55                   	push   %rbp
    114e:	48 89 e5             	mov    %rsp,%rbp
    1151:	48 8d 05 ac 0e 00 00 	lea    0xeac(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
    1158:	48 89 c7             	mov    %rax,%rdi
    115b:	e8 f0 fe ff ff       	call   1050 <puts@plt>
    1160:	b8 00 00 00 00       	mov    $0x0,%eax
    1165:	5d                   	pop    %rbp
    1166:	c3                   	ret

```

在这里不同编译器的行为可能都不一样, 可以在[这里](https://godbolt.org/z/seoG4naT8)
换不同的编译器看结果.
