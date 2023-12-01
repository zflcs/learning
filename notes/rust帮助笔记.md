### Rust 帮助笔记

1. 使用 `link-dead-code` 能够将没有使用的代码连接进目标文件

```rust
[build]
target = "riscv64gc-unknown-none-elf"
target-dir = "../target"

[target.riscv64gc-unknown-none-elf]
rustflags = ["-Clink-args=-Tlinker.ld", "-C", "link-dead-code=yes"]
```

2. 使用 `-Bstatic -l:libc.a` 静态连接 libc

```rust
[build]
target = "riscv64gc-unknown-linux-gnu"
target-dir = "../target"

[target.riscv64gc-unknown-linux-gnu]
linker = "riscv64-unknown-linux-gnu-gcc"
rustflags = ["-Clink-args=-fpie -nostartfiles -Tlinker.ld -Bstatic -l:libc.a"]
```

3. 生成 `riscv64gc-unknown-linux-gnu` 的目标文件时,在 `rustflags` 中加上 `static`

