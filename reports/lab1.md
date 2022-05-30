# 实验一报告
## 总结
> 1. 按照实验书完成了裸机输出 "hello world"
> 2.  实现了彩色输出宏，参考了 rCore 的实现，使用 log 库，可以通过make run LOG=？参数来控制输出等级
> 3. 完成了编程题1、2

## 实验结果

![result](https://github.com/Zfl990109/learning/blob/main/reports/assets/lab1/lab1_result.PNG)

## 问答题
### 1. cpu、寄存器、内存、（外设） 
### 6. 读取当前的 Hart ID CSR mhartid 写入寄存器 a0；将 FDT (Flatten device tree) 在物理内存中的地址写入 a1；跳转到 start_addr ，在实验中是 RustSBI 的地址
> 
	uint32_t reset_vec[10] = {
		0x00000297,                   /* 1:  auipc  t0, %pcrel_hi(fw_dyn) */
		0x02828613,                   /*     addi   a2, t0, %pcrel_lo(1b) */
		0xf1402573,                   /*     csrr   a0, mhartid  */
	#if defined(TARGET_RISCV32)
		0x0202a583,                   /*     lw     a1, 32(t0) */
		0x0182a283,                   /*     lw     t0, 24(t0) */
	#elif defined(TARGET_RISCV64)
		0x0202b583,                   /*     ld     a1, 32(t0) */
		0x0182b283,                   /*     ld     t0, 24(t0) */
		#endif
		0x00028067,                   /*     jr     t0 */
		start_addr,                   /* start: .dword */
		start_addr_hi32,
		fdt_load_addr,                /* fdt_laddr: .dword */
		0x00000000,
									/* fw_dyn: */
	};

### 7. SBI --- 操作系统二进制接口，为操作系统提供硬件服务
### 8. 操作系统和编译器之间达成的协议：？？？
### 10. 操作系统分配空间给应用程序，建立好应用程序执行的环境 
