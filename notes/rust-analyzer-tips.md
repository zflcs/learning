# 关于 rust-analyzer 的一些使用帮助

1. 在修改 riscv 结构的代码时，rust-analyzer 为 x86 架构下代码报错，需要让 rust-analyzer 只检查特定架构的代码
    "rust-analyzer.cargo.target": "riscv64gc-unknown-linux-gnu",
2. 