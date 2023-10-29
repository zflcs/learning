#### Future & Poll



##### Poll 返回值

Poll<uxxxx> 类型大小为普通的 uxxxx 类型的两倍，以 Poll<u32> 为例，它的大小与 u64 相同。Pending 与 Ready 状态的判断根据低 32 位是否为 0 来判断。

- 低 32 位不为 0，则表示 Pending
- 低 32 位为 0，则高 32 为表示 Ready 包裹的返回值

下面的例子展示了 Poll<u32> 与 u64 之间的转换。

```rust
unsafe { log::debug!("{:?}", core::mem::transmute::<Poll<u32>, u64>(Poll::Pending)); } // res: 1
unsafe { log::debug!("{:?}", core::mem::transmute::<u64, Poll<u32>>(0xf00000011)); }   // res: Pending
unsafe { log::debug!("{:?}", core::mem::transmute::<u64, Poll<u32>>(0xf00000000)); }   // res: Ready(15)
```

汇编代码也能佐证：

```asm
00000000802028d4 <_ZN6kernel10test_async28_$u7b$$u7b$closure$u7d$$u7d$17h198b4d6c325432fdE>:
802028d4: 19 71        	addi	sp, sp, -128
802028d6: 86 fc        	sd	ra, 120(sp)
802028d8: a2 f8        	sd	s0, 112(sp)
802028da: a6 f4        	sd	s1, 104(sp)
802028dc: ca f0        	sd	s2, 96(sp)
802028de: ce ec        	sd	s3, 88(sp)
802028e0: d2 e8        	sd	s4, 80(sp)
802028e2: 2a 89        	mv	s2, a0
802028e4: 03 45 05 00  	lbu	a0, 0(a0)						# 猜测 a0 指向的是 Poll 返回值，读取低 8 位的状态
802028e8: 71 ed        	bnez	a0, 0x802029c4 <.Lpcrel_hi82>	# 若不为 0，则表示处于 Pending 状态，跳转到 .Lpcrel_hi82 处
......
00000000802029c4 <.Lpcrel_hi82>:
802029c4: 17 45 00 00  	auipc	a0, 4
802029c8: 13 05 c5 a1  	addi	a0, a0, -1508

00000000802029cc <.Lpcrel_hi83>:
802029cc: 97 45 00 00  	auipc	a1, 4
802029d0: 13 86 45 a7  	addi	a2, a1, -1420
802029d4: 93 05 30 02  	li	a1, 35
802029d8: 97 20 00 00  	auipc	ra, 2
802029dc: e7 80 80 b0  	jalr	-1272(ra)			# 猜测是返回到上一层的 Poll 函数
802029e0: 00 00        	unimp
```





##### async 函数产生异常

###### async 函数主体

```rust
#[no_mangle]
async fn test_async() -> isize {
    println!("test ok!!!!");
    unsafe { core::arch::asm!("ebreak"); }		// 产生异常，跳转到到 stvec 寄存器指向的异常处理函数
    let a = 3 + 3;
    println!("test ok!!!! {}", a);
    0
}
```

###### stvec 异常处理函数

```rust
#[naked]
#[no_mangle]
#[link_section = ".text.trampoline"]
pub extern "C" fn trampoline() {
    unsafe {
        core::arch::asm!(
            // "csrr tp, sepc",
            // "addi tp, tp, 2",
            // "csrw sepc, tp",
            // "sret",
            "addi ra, ra, 2",
            "ret",
            options(noreturn)
        )
    }
}
```

直接将 ra 或者 sepc 寄存器移动到下一条代码位置，ret 或 sret 都会让协程继续执行，这里需要进一步找到 async 的上层调用者的入口，这里需要进一步理解 rust 编译器对 async 的处理，找到正确的返回地址。并且需要在运行时，能够找到对应的返回地址。



```rust
#[naked]
#[no_mangle]
#[link_section = ".text.trampoline"]
pub extern "C" fn trampoline() {
    unsafe {
        core::arch::asm!(
            "ld ra, 376(sp)",				// 这里先通过反汇编获取上述 async 函数执行时，sp 指针的变化
            "addi sp, sp, 384",				// 再进行硬编码，因此需要动态获取 sp 的变化
            "ret",
            options(noreturn)
        )
    }
}
```

在 trampoline 中模拟 async 函数调用返回，将 ra 和 sp 恢复之后，能够正确回复到上层的调用者的 poll 函数，并且还可以重复进入到 async 函数，只是会将 async 函数上一次已经执行的代码重复执行。本来异常产生后，只需要回退到产生异常的那条指令，只需要回退一小步，在这里相当于回退了一大步。结果如下所示。其中 打印 test ok！表示重复进入了 async 函数。

```
test ok!
[DEBUG] executor/src/task.rs:52 Ready ok
test ok!
[DEBUG] executor/src/task.rs:52 Ready ok
```



总体来说，还是需要进一步跟踪 sp、ra 寄存器的变化，这里可能需要参考 retprobe。



上述的构造 Poll 函数返回值的办法，由于需要准确的定位到上层的调用函数，需要根据函数的返回值进行 probe，这里还需要进一步的讨论。还可以有以下两种方式：

- 构造一个假的协程，让它来让出 CPU，与方法 1 没有实质性的区别
- 保存寄存器，切换到新准备的栈（或直接复用栈），并且运行 poll 函数，让其余的协程能够继续运行，这里不需要理解编译器的处理过程



