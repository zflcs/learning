
### qemu 学习笔记

#### 官方文档上的一些介绍

1. 对于官网上未列出的 host 架构，可以使用 --enable-tcg-interpreter 打开 TCI 支持（但不建议这么做）
2. 使用 arg_name=on 形式传递 TCG 插件参数
3. pmu-num=n 支持更灵活的 counter 配置
4. loader 设备支持将数据、文件放在指定位置


#### 源码分析

先从 target/riscv 目录开始阅读

```
.
├── arch_dump.c             // 在 core dump 时提供信息
├── bitmanip_helper.c       // RISC-V 位操作扩展
├── common-semi-target.h
├── cpu_bits.h              // RISC-V 架构常量
├── cpu.c                   // 定义与 CPU 状态相关的函数
├── cpu_cfg.h
├── cpu.h                   // 一些与 CPU 状态相关的数据结构
├── cpu_helper.c
├── cpu-param.h
├── cpu-qom.h
├── cpu_user.h
├── cpu_vendorid.h
├── crypto_helper.c
├── csr.c                   // CSR 寄存器操作
├── debug.c
├── debug.h
├── fpu_helper.c
├── gdbstub.c
├── helper.h
├── insn16.decode
├── insn32.decode
├── insn_trans              // 指令翻译（根据 RISC-V 扩展进行分类）
├── instmap.h               // 指令解码相关
├── internals.h
├── Kconfig
├── kvm                 // KVM 相关
├── m128_helper.c
├── machine.c
├── meson.build
├── monitor.c
├── op_helper.c
├── pmp.c
├── pmp.h
├── pmu.c
├── pmu.h
├── riscv-qmp-cmds.c
├── sbi_ecall_interface.h
├── tcg
│   ├── meson.build
│   ├── tcg-cpu.c
│   └── tcg-cpu.h
├── time_helper.c
├── time_helper.h
├── trace-events
├── trace.h
├── translate.c
├── vcrypto_helper.c
├── vector_helper.c
├── vector_internals.c
├── vector_internals.h
├── xthead.decode
├── XVentanaCondOps.decode
└── zce_helper.c
```

#### 添加虚拟设备

参考这篇[资料](https://zhuanlan.zhihu.com/p/665797722) 


##### 头文件

首先定义设备类型，通常是一个宏表示字符串。

OBJECT_CHECK: 将 Object 对象转化成自定义的设备对象
    OBJECT_CHECK -> object_dynamic_cast_assert

qdev_prop_set_uint32: 设置对象属性




