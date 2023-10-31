#### Rust async can truly be zero-cost

[Rust async can truly be zero-cost (swatinem.de)](https://swatinem.de/blog/zero-cost-async/)



它使用了一个简单的例子来解释 Rust 编译器如何处理 async/await 关键字。具体的细节见 [Compiler Explorer (godbolt.org)](https://godbolt.org/)。



```rust
async fn do_stuff<P: StuffProvider>(mut provider: P) -> Stuff {
    let mut stuff = provider.provide_stuff().await;
    stuff.0 += 1;
    stuff
}
```

上述 await 关键字转化成汇编代码后，具体表现为跳转到 .LBB0_5 或 .LBB0_7 处，恢复 sp 指针并返回到上层调用者

```assembly
core::ptr::drop_in_place<example::do_stuff<example::SyncStuffProvider>::{{closure}}>:
        addi    sp, sp, -32
        lbu     a0, 0(a0)
        sd      a0, 8(sp)
        beqz    a0, .LBB0_3
        j       .LBB0_1
.LBB0_1:
        ld      a0, 8(sp)
        sext.w  a0, a0
        li      a1, 3
        beq     a0, a1, .LBB0_4
        j       .LBB0_2
.LBB0_2:
        addi    sp, sp, 32
        ret
.LBB0_3:
        j       .LBB0_5
.LBB0_4:
        j       .LBB0_6
.LBB0_5:
        addi    sp, sp, 32
        ret
.LBB0_6:
        j       .LBB0_7
.LBB0_7:
        addi    sp, sp, 32
        ret
```



此外，作者还指出，在某种特定情况下 [Issue #71093](https://github.com/rust-lang/rust/issues/71093) ，编译器无法为我们编译出简单的代码。这种情况是因为这个 async 函数内部可能 panic，与重复 poll 已经完成的 async 函数相关。作者提出可以使用 [#[no_panic] 属性宏](https://github.com/dtolnay/no-panic)，让其不能通过编译。