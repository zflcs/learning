# 实验二报告

### 编程题1

```
use core::{arch::asm, ptr};
#[inline(always)]
pub fn print_stack_trace() {
    // 获取到栈帧指针
    let mut fp: *const usize;
    unsafe {
        asm!(
            "mv {fp}, fp",
            fp = out(reg) fp,
        );
        log::trace!("=== Stack trace from fp chain ===");
        while fp != ptr::null() {

            log::trace!("  ra:{:#x}  fp: {:#x}", *(fp.sub(1)), *(fp.sub(2)));
            fp = *(fp.sub(2)) as *const usize;
        }
        log::trace!("=== End ===\n\n");
    }
}
```

### 编程题2、3、4之后再补充

### 问答题

```
1. 函数调用与系统调用的区别？
	函数调用是普通的控制流，不会涉及特权级转换，而系统调用则会使用专门的指令，切换到内核态，涉及到特权级转换
2. M 态软件会将 S 态异常/中断委托给 S 态软件，请指出有哪些寄存器记录了委托信息，rustsbi 委托了哪些异常/中断？
	mideleg 寄存器记录了中断委托， ssoft（S 模式软中断）、stimer（S 模式时钟中断）、sext（S 模式外部中断）
	medeleg 寄存器记录了异常委托，ima（指令未对齐）、ia（取指访问异常）、bkpt（断点）、la（读异常）、sa（写异常）、uecall（U 模式系统调用）、ipage（取指令 page fault）、lpage（读 page fault）、spage（写 page fault）
3. 如果操作系统以应用程序库的形式存在，应用程序可以通过哪些方式破坏操作系统？
4. 编译器/操作系统/处理器如何合作，可采用哪些方法来保护操作系统不受应用程序的破坏？
	编译器在编译时尽量拒绝不安全的指令，操作系统提供虚拟环境来给应用程序运行，处理器结合特定的指令集，设置特权级
5. RISC-V处理器的S态特权指令有哪些，其大致含义是什么，有啥作用？
	ecall：发出 S 模式的中断或异常；sret：返回 U 模式；
6. RISC-V处理器在用户态执行特权指令后的硬件层面的处理过程是什么？
	自动保存 sstatus 、sepc 等寄存器，然后跳转到 stvec 中的中断处理函数
7. 操作系统在完成用户态<–>内核态双向切换中的一般处理过程是什么？
	主要是保存和恢复上下文
8. 程序陷入内核的原因有中断、异常和陷入（系统调用），请问 riscv64 支持哪些中断 / 异常？如何判断进入内核是由于中断还是异常？描述陷入内核时的几个重要寄存器及其值。
	具体支持的异常和中断，参见 RISC-V 特权集规范 The RISC-V Instruction Set Manual Volume II: Privileged Architecture 。其它很多问题在这里也有答案。
	scause 的最高位，为 1 表示中断，为 0 表示异常
	重要的寄存器：
		scause ：发生了具体哪个异常或中断
		sstatus ：其中的一些控制为标志发生异常时的处理器状态，如 sstatus.SPP 表示发生异常时处理器在哪个特权级。
		sepc ：发生异常或中断的时候，将要执行但未成功执行的指令地址
		stval ：值与具体异常相关，可能是发生异常的地址，指令等
9. 在哪些情况下会出现特权级切换：用户态–>内核态，以及内核态–>用户态？
	内核态->用户态：1. 第一次执行应用程序，2. 中断或异常
	用户态到内核态：1. 中断或异常 2. 应用程序执行结束（实际上也是通过系统调用 exit()）
```

