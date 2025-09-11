---
title: RISC-V RVV intrinsic指令优化指南
date: 2025-09-10 22:31:55
categories: RISC-V
tags: [RISC-V, RVV]
---

## 引言

RISC-V的Vector扩展使得RISC-V处理器能够执行向量计算，

<!-- ## Related Issues -->
<!---->
<!-- - [clangd#2143](https://github.com/clangd/clangd/issues/2143) -->
<!-- - [llvm-project#109634](https://github.com/llvm/llvm-project/issues/109634) -->
<!---->
<!-- The RISC-V vector intrinsic functions are defined in `<riscv_vector.h>` as follows: -->
<!---->
<!-- ```c -->
<!-- #pragma clang riscv intrinsic vector -->
<!-- ``` -->
<!---->
<!-- While the compiler can recognize these, `clangd` currently does not. It may support them in the future, but for now, you'll need to manually generate these header files as a workaround. -->
<!---->
<!-- ## Demo -->
<!---->
<!-- Let's write some RVV code! -->
<!---->
<!-- ![demo](https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExNDg0dW1zandwNzFkZGoyNWp3YTZlMnZwdzZ0aDYxdHhyaTZraWhkZSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/lQUGbpbTKQBWVulhxS/giphy.gif) -->
<!---->
<!-- ## How to Use -->
<!---->
<!-- ### 1. Generate Header Files -->
<!---->
<!-- Start by generating the intrinsic files from `rvv-intrinsic-doc`. Clone the submodule: -->
<!---->
<!-- ```bash -->
<!-- git submodule update --init --recursive -->
<!-- ``` -->
<!---->
<!-- The intrinsic version is v0.12.x. Prepare your Python environment, navigate to the `rvv-intrinsic-doc` directory, and then run the following commands: -->
<!---->
<!-- ```shell -->
<!-- cd rvv-intrinsic-generator -->
<!-- pip install -r requirements.txt -->
<!-- make -->
<!-- ``` -->
<!---->
<!-- The intrinsic files will be generated in `rvv-intrinsic-doc/auto-generated/` in `.md` format. We need to convert these to `.h` format. -->
<!---->
<!-- Run the script in the `scripts` directory: -->
<!---->
<!-- ```shell -->
<!-- ./scripts/run.py -->
<!-- ``` -->
<!---->
<!-- This will generate the header files in the `rvv_intrinsic` directory. -->
<!---->
<!-- ### 2. Configure Clangd -->
<!---->
<!-- The `rvv_intrinsic.h` file organizes the header files with corresponding extensions. You do not need to include this file directly in your application, as the compiler already knows about these intrinsics. -->
<!---->
<!-- However, `clangd` does not automatically recognize them. To configure `clangd`, please refer to the [Configurations](https://clangd.llvm.org/config) documentation. Add the following configuration to your `.clangd` file: -->
<!---->
<!-- ```yaml -->
<!-- CompileFlags: -->
<!--   Add: -->
<!--     [ -->
<!--       -include /path/to/repo/rvv_intrinsic/rvv_intrinsic.h, -->
<!--       -I /path/to/repo/rvv-intrinsic/rvv_intrinsic, -->
<!--     ] -->
<!-- ``` -->
<!---->
<!-- This will force the inclusion of `rvv_intrinsic.h` and allow `clangd` to locate the generated header files. -->
<!---->
<!-- ## Testing -->
<!---->
<!-- Before running tests, make sure you have prepared the RISC-V toolchain and `bear`. `bear` is used to generate the `compile_commands.json` compilation database, which is required by `clangd`. -->
<!---->
<!-- - [RISC-V GNU Toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) -->
<!-- - [Bear](https://github.com/rizsotto/Bear) -->
<!---->
<!-- The `tests` directory contains various RISC-V vector extension subset test cases, including: -->
<!---->
<!-- - _zve32x -->
<!-- - _zve32f -->
<!-- - _zfh_zvfh_zve32f -->
<!-- - _zve64x -->
<!-- - _zve64f -->
<!-- - _zve64d -->
<!---->
<!-- For more details on the different vector extensions, refer to the [RISC-V V-Spec v1.0](https://github.com/riscvarchive/riscv-v-spec/releases/tag/v1.0). -->
<!---->
<!-- To generate the compilation database (`compile_commands.json`), run the following command in the directory where the `Makefile` exists: -->
<!---->
<!-- ```shell -->
<!-- bear -- make -->
<!-- ``` -->
<!---->
<!-- If you have the `clangd` [plugin](https://clangd.llvm.org/installation#editor-plugins) installed in VSCode or Neovim, it should work seamlessly. -->
<!---->
<!-- --- -->
<!---->
<!-- Let me know if you'd like to adjust anything further! -->
<!---->
<!-- ## License -->
<!---->
<!-- This project is licensed under the Apache License 2.0 - see the [LICENSE.txt](./LICENSE.txt) file for details. -->
<!---->
