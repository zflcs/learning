### windows开发环境搭建

1. #### 安装windows rust

    1. 安装 Rust 版本管理器 rustup 和 Rust 包管理器 cargo（[入门 - Rust 程序设计语言 (rust-lang.org)](https://www.rust-lang.org/zh-CN/learn/get-started)）

2. #### 安装 c 环境支持

    1. 安装 c++ 生成工具（[Microsoft C++ 生成工具 - Visual Studio](https://visualstudio.microsoft.com/zh-hans/visual-cpp-build-tools/)）
    2. 勾选 c++ 桌面开发

3. #### 安装 windows qemu

    1. 安装最新版本qemu，默认安装即可[QEMU for Windows – Installers (64 bit) (weilnetz.de)](https://qemu.weilnetz.de/w64/)
    2. 在系统环境变量中添加 qemu 的安装目录
    3. 在 cmd 中输入 `qemu-system-riscv64 --version` 可以显示 qemu 版本

4. #### 下载 代码

    ```
    git clone https://github.com/rcore-os/rCore-Tutorial-v3
    ```

5. #### 编写 python 脚本编译运行

    1. 编写脚本 build.rs，对每章的 makefile 编译脚本以及启动 qemu 用 python 实现

```python
import os

os.system("rustup target add riscv64gc-unknown-none-elf")
os.system("cargo install cargo-binutils")
os.system("rustup component add llvm-tools-preview")
os.system("rustup component add rust-src")
os.system("cargo clean")

os.system("cd user && cargo build --release")

os.system("cd easy-fs-fuse && cargo run --release -- -s ../user/src/bin/ -t ../user/target/riscv64gc-unknown-none-elf/release/")

os.system("cd os && cargo build --release")

os.system("qemu-system-riscv64 \
-machine virt \
-nographic \
-smp cpus=1 \
-bios bootloader/rustsbi-qemu.bin \
-device loader,file=os/target/riscv64gc-unknown-none-elf/release/os,addr=0x80200000 \
-drive file=user/target/riscv64gc-unknown-none-elf/release/fs.img,if=none,format=raw,id=x0 \
-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0")
```



