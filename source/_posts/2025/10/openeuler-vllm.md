---
title: openEuler基础配置与vllm的初步探索
date: 2025-10-26 15:45:36
tags:
  - openEuler
  - RISC-V
  - vllm
categories:
  - RISC-V
  - openEuler
---

今天又折腾了一下openEuler和vllm.

## openEuler RISC-V QEMU镜像

openEuler的镜像是官方提供的, 今天意识到原来`qemu-img`工具如此强大.

一方面是镜像可以增量保存, 通过`qemu-img create`创建一个新的镜像, 不会修改原有的基础镜像.
者可以很方便得在不同版本的os之间切换.

```bash
qemu-img create -o backing_file=base.qcow2,backing_fmt=qcow2 -f qcow2 test.qcow2
```

另外QEMU的镜像也可以创建snapshot, 查看已有的snapshot, 从snapshot中恢复镜像

```bash
qemu-img snapshot -c snap1 test.qcow2
qemu-img snapshot -l test.qcow2
qemu-img snapshot -a snap1 test.qcow2
```

## 创建管理用户

刚进入系统时只有root用户, 为了后面开发更安全, 还是有必要先创建一个普通用户

```bash
useradd <username>
passwd <username>
```

改一下`/etc/sudoers`, 给`<username>`加上sudo权限

再装一个`zsh`, 改一下`/etc/passwd`设置一下默认的shell为`zsh`, 方便用zsh的autosuggestion和highlight

## 安装软件

openEuler推荐用`dnf`来管理软件包, 用下面的命令可以列出所有的软件源.

```bash
dnf repolist all
```

据openEuler的文档介绍, 只需要留`OS`和`source`两个软件源就可以了,
其它软件源可以通过`dnf config-manager`来启用或禁用

```bash
dnf config-manager --enable/disable <repo>
```

安装, 卸载, 搜索软件包和`apt`差不多, 都是`install`, `remove`, `search`命令

## vllm

vllm是一个python的包, 可以用来做语言模型的推理, 我先尝试在x86的机器上跑,
结果提示我的GPU显存不足, 所以还是需要根据源码来编译一个版本进行测试.

我没有用`uv`管理环境, 而是用`conda`, 在创建conda虚拟环境之后安装依赖

```bash
pip install -r requirements/cpu-build.txt
pip install -r requirements/cpu.txt
```

编译安装`vllm`

```bash
VLLM_TARGET_DEVICE=cpu uv pip install . --no-build-isolation
```

这样就成功安装来CPU版本的vllm了, 我用的源码是`v0.11.0`版本的, 也就是最新的release版本,
前面`0.9.2`版本好像有点问题.

这样就可以跑指定的模型来做性能测试了:

```bash
vllm bench throughput --model "Qwen/Qwen2.5-7B-Instruct" --input-len 64 --output-len 64 --num-prompts 1
```

如果要在openEuler RISC-V的虚拟机上编译`vllm`的话感觉有点麻烦, 需要之后再尝试一下.

## 参考文档

- [vllm cli](https://docs.vllm.ai/en/latest/cli/bench/throughput.html)
- [vllm cpu](https://docs.vllm.ai/en/latest/getting_started/installation/cpu.html)
