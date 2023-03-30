
### FPGA 部署流程

目前已经能够在 WSL 上运行模拟器，看到串口得输出了。

#### 构建 vivado 项目
按照[标签化 RISC-V fpga部署指南](https://github.com/LvNA-system/labeled-RISC-V/blob/master/fpga/README.md#run-with-fpga)的提示进行操作，目前仓库里已经增加了 zcu102 的补丁。

因为 vivado 安装在 windows 上，之前曾经在 wsl 上启动过 windows 下的 vivado，所以按照这个思路修改环境变量试图直接在 wsl 中启动 windows 下的 vivado，没有成功。

所以还是按照提示来进行，先在 lrv 的 fpga 目录中执行一遍 `make -C pardcore BOARD=zcu102` 指令，生成 pardcore，再下载到 windows 上。

vivado2022.2 上支持的 zcu102 的版本号是 3.4，而代码中的版本是 3.3，需要将 mk.tcl 中的版本号改成 3.4，再进行尝试。

改完执行脚本之后，发现仍然报错 3.3 版本不对，发现 board/zcu102/bd/prm.tcl 中有两个硬编码 3.3，改成 3.4

产生了一堆警告

```
PS E:\fpga> vivado -nolog -nojournal -notrace -mode batch -source board/zcu102/mk.tcl -tclargs myproject-zcu102

****** Vivado v2022.2 (64-bit)
  **** SW Build 3671981 on Fri Oct 14 05:00:03 MDT 2022
  **** IP Build 3669848 on Fri Oct 14 08:30:02 MDT 2022
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.

source board/zcu102/mk.tcl -notrace
INFO: [BD_TCL-3] Currently there is no design <pardcore> in project, so creating one...
Wrote  : <E:\fpga\board\zcu102\build\myproject-zcu102\myproject-zcu102.srcs\sources_1\bd\pardcore\pardcore.bd>
INFO: [BD_TCL-4] Making design <pardcore> as current_bd_design.
INFO: [BD_TCL-5] Currently the variable <design_name> is equal to "pardcore".
INFO: [BD_TCL-6] Checking if the following IPs exist in the project's IP catalog:  xilinx.com:ip:c_shift_ram:12.0 xilinx.com:ip:xlconstant:1.1 xilinx.com:ip:xlslice:1.0  .
INFO: [BD_TCL-6] Checking if the following modules exist in the project's sources:  LvNAFPGATop  .
WARNING: [BD 5-670] It is required to provide a frequency value for a user created input clock port. Please use the <-freq_hz $freq_val> argument of the create_bd_port command. ie create_bd_port -dir I -type clk -freq_hz 100000000 clkin
INFO: [IP_Flow 19-5151] The Range '63:0' is present in all ports of the interface 'ila_csr_rw'. It is assumed that this is meant to declare an array of interface. If this is not the desired behaviour, switch of this feature by disabling the parameter 'ips.enableInterfaceArrayInference'.
INFO: [IP_Flow 19-5107] Inferred bus interface 'l2_frontend_bus_axi4_0' of definition 'xilinx.com:interface:aximm:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-5107] Inferred bus interface 'mem_axi4_0' of definition 'xilinx.com:interface:aximm:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-5107] Inferred bus interface 'mmio_axi4_0' of definition 'xilinx.com:interface:aximm:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-5107] Inferred bus interface 'debug_systemjtag_reset' of definition 'xilinx.com:signal:reset:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-5107] Inferred bus interface 'reset' of definition 'xilinx.com:signal:reset:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-5107] Inferred bus interface 'reset_to_hang_en' of definition 'xilinx.com:signal:reset:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-5107] Inferred bus interface 'clock' of definition 'xilinx.com:signal:clock:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-4728] Bus Interface 'clock': Added interface parameter 'ASSOCIATED_BUSIF' with value 'l2_frontend_bus_axi4_0'.
INFO: [IP_Flow 19-4728] Bus Interface 'clock': Added interface parameter 'ASSOCIATED_RESET' with value 'reset'.
INFO: [IP_Flow 19-234] Refreshing IP repositories
INFO: [IP_Flow 19-1704] No user IP repositories specified
INFO: [xilinx.com:ip:c_shift_ram:12.0-913] /c_shift_ram_1 Width has been set to manual on the GUI. It will not be updated during validation with a propagated value.
INFO: [xilinx.com:ip:c_shift_ram:12.0-913] /c_shift_ram_0 Width has been set to manual on the GUI. It will not be updated during validation with a propagated value.
WARNING: [BD 41-237] Bus Interface property AWUSER_WIDTH does not match between /LvNAFPGATop_0/l2_frontend_bus_axi4_0(0) and /S_AXI_DMA(5)
WARNING: [BD 41-237] Bus Interface property ARUSER_WIDTH does not match between /LvNAFPGATop_0/l2_frontend_bus_axi4_0(0) and /S_AXI_DMA(5)
WARNING: [BD 41-237] Bus Interface property WUSER_WIDTH does not match between /LvNAFPGATop_0/l2_frontend_bus_axi4_0(0) and /S_AXI_DMA(5)
WARNING: [BD 41-237] Bus Interface property RUSER_WIDTH does not match between /LvNAFPGATop_0/l2_frontend_bus_axi4_0(0) and /S_AXI_DMA(5)
WARNING: [BD 41-237] Bus Interface property BUSER_WIDTH does not match between /LvNAFPGATop_0/l2_frontend_bus_axi4_0(0) and /S_AXI_DMA(5)
CRITICAL WARNING: [BD 41-759] The input pins (listed below) are either not connected or do not have a source port, and they don't have a tie-off specified. These pins are tied-off to all 0's to avoid error in Implementation flow.
Please check your design and connect them as needed:
/LvNAFPGATop_0/l2_frontend_bus_axi4_0_awinstret
/LvNAFPGATop_0/l2_frontend_bus_axi4_0_arinstret

Wrote  : <E:\fpga\board\zcu102\build\myproject-zcu102\myproject-zcu102.srcs\sources_1\bd\pardcore\pardcore.bd>
INFO: [BD::TCL 103-2003] Currently there is no design <zynq_soc> in project, so creating one...
Wrote  : <E:\fpga\board\zcu102\build\myproject-zcu102\myproject-zcu102.srcs\sources_1\bd\zynq_soc\zynq_soc.bd>
INFO: [BD::TCL 103-2004] Making design <zynq_soc> as current_bd_design.
INFO: [BD::TCL 103-2005] Currently the variable <design_name> is equal to "zynq_soc".
INFO: [BD::TCL 103-2011] Checking if the following IPs exist in the project's IP catalog:  xilinx.com:ip:xlconcat:2.1 xilinx.com:ip:zynq_ultra_ps_e:3.4 xilinx.com:ip:clk_wiz:6.0 xilinx.com:ip:proc_sys_reset:5.0 xilinx.com:ip:axi_dma:7.1 xilinx.com:ip:axi_gpio:2.0 xilinx.com:ip:xlslice:1.0 xilinx.com:ip:axi_crossbar:2.1 xilinx.com:ip:axi_uart16550:2.0 xilinx.com:ip:axi_uartlite:2.0 xilinx.com:ip:axi_clock_converter:2.1  .
INFO: [BD::TCL 103-2020] Checking if the following modules exist in the project's sources:  axi_jtag_v1_0  .
create_bd_cell: Time (s): cpu = 00:00:04 ; elapsed = 00:00:19 . Memory (MB): peak = 2224.555 ; gain = 1032.250
INFO: [IP_Flow 19-5107] Inferred bus interface 's_axi' of definition 'xilinx.com:interface:aximm:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-5107] Inferred bus interface 's_axi_aresetn' of definition 'xilinx.com:signal:reset:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-5107] Inferred bus interface 's_axi_aclk' of definition 'xilinx.com:signal:clock:1.0' (from Xilinx Repository).
INFO: [IP_Flow 19-4728] Bus Interface 's_axi_aresetn': Added interface parameter 'POLARITY' with value 'ACTIVE_LOW'.
INFO: [IP_Flow 19-4728] Bus Interface 's_axi_aclk': Added interface parameter 'ASSOCIATED_BUSIF' with value 's_axi'.
INFO: [IP_Flow 19-4728] Bus Interface 's_axi_aclk': Added interface parameter 'ASSOCIATED_RESET' with value 's_axi_aresetn'.
INFO: [IP_Flow 19-234] Refreshing IP repositories
INFO: [IP_Flow 19-1704] No user IP repositories specified
WARNING: [BD 41-1306] The connection to interface pin </hier_prm_peripheral/GPIO_NOHYPE_SETTINGS/gpio_io_o> is being overridden by the user with net <GPIO_NOHYPE_SETTINGS_gpio_io_o>. This pin will not be connected as a part of interface connection <GPIO>.
WARNING: [BD 41-1306] The connection to interface pin </hier_prm_peripheral/pardcore_corerst/gpio_io_o> is being overridden by the user with net <axi_gpio_0_gpio_io_o>. This pin will not be connected as a part of interface connection <GPIO>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uart16550_pardcore_1/sout> is being overridden by the user with net <axi_uart16550_1_sout>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uartlite_1/rx> is being overridden by the user with net <axi_uart16550_1_sout>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uart16550_pardcore_2/sout> is being overridden by the user with net <axi_uart16550_2_sout>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uartlite_2/rx> is being overridden by the user with net <axi_uart16550_2_sout>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uart16550_pardcore_3/sout> is being overridden by the user with net <axi_uart16550_3_sout>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uart16550_pardcore_4/sin> is being overridden by the user with net <axi_uart16550_3_sout>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uart16550_pardcore_3/sin> is being overridden by the user with net <axi_uart16550_4_sout>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uart16550_pardcore_4/sout> is being overridden by the user with net <axi_uart16550_4_sout>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uartlite_0/tx> is being overridden by the user with net <axi_uartlite_0_tx>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uartlite_pardcore_0/rx> is being overridden by the user with net <axi_uartlite_0_tx>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uart16550_pardcore_1/sin> is being overridden by the user with net <axi_uartlite_1_tx>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uartlite_1/tx> is being overridden by the user with net <axi_uartlite_1_tx>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uart16550_pardcore_2/sin> is being overridden by the user with net <axi_uartlite_2_tx>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uartlite_2/tx> is being overridden by the user with net <axi_uartlite_2_tx>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uartlite_0/rx> is being overridden by the user with net <axi_uartlite_pardcore_0_tx>. This pin will not be connected as a part of interface connection <UART>.
WARNING: [BD 41-1306] The connection to interface pin </hier_uart/axi_uartlite_pardcore_0/tx> is being overridden by the user with net <axi_uartlite_pardcore_0_tx>. This pin will not be connected as a part of interface connection <UART>.
Slave segment '/hier_prm_peripheral/GPIO_NOHYPE_SETTINGS/S_AXI/Reg' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x8000_4000 [ 4K ]>.
Slave segment '/M_AXI_SBUS/Reg' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x10_0000_0000 [ 64G ]>.
Slave segment '/hier_prm_peripheral/axi_dma_arm/S_AXI_LITE/Reg' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x8002_0000 [ 4K ]>.
Slave segment '/hier_prm_peripheral/axi_jtag_v1_0_0/s_axi/reg0' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x8001_1000 [ 4K ]>.
Slave segment '/hier_uart/axi_uartlite_0/S_AXI/Reg' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x8000_0000 [ 4K ]>.
Slave segment '/hier_uart/axi_uartlite_1/S_AXI/Reg' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x8000_1000 [ 4K ]>.
Slave segment '/hier_uart/axi_uartlite_2/S_AXI/Reg' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x8000_2000 [ 4K ]>.
Slave segment '/hier_uart/axi_uartlite_3/S_AXI/Reg' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x8000_3000 [ 4K ]>.
Slave segment '/hier_prm_peripheral/pardcore_corerst/S_AXI/Reg' is being assigned into address space '/zynq_ultra_ps_e_0/Data' at <0x8001_0000 [ 4K ]>.
Slave segment '/M_AXI_DMA/Reg' is being assigned into address space '/hier_parcore_peripheral/axi_dma_pardcore/Data_SG' at <0x000_0000_0000 [ 1T ]>.
Slave segment '/M_AXI_DMA/Reg' is being assigned into address space '/hier_parcore_peripheral/axi_dma_pardcore/Data_MM2S' at <0x000_0000_0000 [ 1T ]>.
Slave segment '/M_AXI_DMA/Reg' is being assigned into address space '/hier_parcore_peripheral/axi_dma_pardcore/Data_S2MM' at <0x000_0000_0000 [ 1T ]>.
Slave segment '/zynq_ultra_ps_e_0/SAXIGP2/HP0_DDR_LOW' is being assigned into address space '/hier_prm_peripheral/axi_dma_arm/Data_SG' at <0x0000_0000 [ 2G ]>.
Slave segment '/zynq_ultra_ps_e_0/SAXIGP2/HP0_DDR_LOW' is being assigned into address space '/hier_prm_peripheral/axi_dma_arm/Data_MM2S' at <0x0000_0000 [ 2G ]>.
Slave segment '/zynq_ultra_ps_e_0/SAXIGP2/HP0_DDR_LOW' is being assigned into address space '/hier_prm_peripheral/axi_dma_arm/Data_S2MM' at <0x0000_0000 [ 2G ]>.
Slave segment '/hier_parcore_peripheral/axi_dma_pardcore/S_AXI_LITE/Reg' is being assigned into address space '/S_AXI_MMIO' at <0x6001_0000 [ 4K ]>.
Slave segment '/hier_uart/axi_uart16550_pardcore_1/S_AXI/Reg' is being assigned into address space '/S_AXI_MMIO' at <0x6000_1000 [ 4K ]>.
Slave segment '/hier_uart/axi_uart16550_pardcore_2/S_AXI/Reg' is being assigned into address space '/S_AXI_MMIO' at <0x6000_2000 [ 4K ]>.
Slave segment '/hier_uart/axi_uart16550_pardcore_3/S_AXI/Reg' is being assigned into address space '/S_AXI_MMIO' at <0x6000_3000 [ 4K ]>.
Slave segment '/hier_uart/axi_uart16550_pardcore_4/S_AXI/Reg' is being assigned into address space '/S_AXI_MMIO' at <0x6000_4000 [ 4K ]>.
Slave segment '/hier_uart/axi_uartlite_pardcore_0/S_AXI/Reg' is being assigned into address space '/S_AXI_MMIO' at <0x6000_0000 [ 4K ]>.
Slave segment '/zynq_ultra_ps_e_0/SAXIGP2/HP0_DDR_LOW' is being assigned into address space '/S_AXI_MEM' at <0x8_0000_0000 [ 2G ]>.
Wrote  : <E:\fpga\board\zcu102\build\myproject-zcu102\myproject-zcu102.srcs\sources_1\bd\zynq_soc\zynq_soc.bd>
WARNING: [BD::TCL 103-2053] This Tcl script was generated from a block design that has not been validated. It is possible that design <zynq_soc> may result in errors during validation.
INFO: [Common 17-206] Exiting Vivado at Thu Mar 30 11:23:34 2023...
```

但是生成了 vivado 项目，进行综合之后报了一堆错误，查看 errors，发现是因为 license 的原因，尝试获取 license。

直接点击生成比特流，自动运行了 implement 和 synthesis，但是在 synthesis 时出现了某些问题，查看 errors 发现，是因为内存用完了，在进行了多次综合子模块之后，总算把所有的模块综合出来了。

生成比特流时产生了警告

![](./assets/genbit_warn.png)

最终成功生成了 bit 和 bin 文件。

#### shell 响应慢
初步怀疑和内核目前的调度方式相关，内核的调度以协程的形式来完成，因此在切换回原来的形式（初始化之后运行调度函数，只有核 0 处理内核协程）之后，shell 的响应速度正常。

将内核的调度实现为协程之后，内核初始化完成之后，处于 idle 线程（没有显式的初始化创建线程操作）内，执行循环 poll 内核协程的函数（poll_kernel_future），与用户线程相同，但内核线程的状态转换与用户线程的状态转换存在区别。

##### 目前调度协程的实现
```rust
pub async fn run_tasks() {
    let mut helper = Box::new(ReadHelper::new());
    loop {
        if let Some(task) = fetch_task() {
            PROCESSORS[hart_id()].run_next(task);
            PROCESSORS[hart_id()].suspend_current();
        }
		helper.as_mut().await;
    }
}
```
无论能否从线程池取出协程，都会对 helper 进行 poll，这导致了大量的协程切换开销。假设在执行某个线程之后，它会向内核提交某些内核协程，这些协程的优先级比调度协程的优先级高，因此在发生时钟中断回到 idle 线程时，内核的调度协程需要让权。基于上述的假设，将 await 放在 if 语句内，减少协程切换开销。
```rust
pub async fn run_tasks() {
    let mut helper = Box::new(ReadHelper::new());
    loop {
        if let Some(task) = fetch_task() {
            PROCESSORS[hart_id()].run_next(task);
            PROCESSORS[hart_id()].suspend_current();
			helper.as_mut().await;
        }
    }
}
```

移动到 if 语句内，仍然不能即使响应，在将切换的频率降低之后，得到了正常响应。

#### 0x101000000 页错误

把 MEMORY_END 扩大之后，遇到了 0x101000000 页错误，还不清楚是什么原因。

所有用户进程也同时映射了 trace 阶段的地址，因此在将所有的 trace 代码注释之后，能够正常运行测试。

堆进行了扩展，目前应该可以运行相关的测试。

#### 创建进程时，查找符号表慢（暂时不考虑）



#### apmr（环个数+环长度+数据量+线程数）
##### 4 + 512 + 1 + 1
8371、8532、8344、8371、8415、8451、8402、8535、8435、8343
##### 4 + 512 + 1 + 2
5804、5875、5745、5776、5707、5894、5795、5692、5817、5900
##### 4 + 512 + 1 + 3
4263、4315、4305、4298、4323、4284、4334、4369、4300、4337
##### 4 + 512 + 1 + 4
4264、4197、4237、4278、4276、4290、4262、4237、4227、4246
##### 4 + 512 + 1 + 5
18871、19873、19902、14465、20403、19302、19302、20289、18357、18938

##### 4 + 512 + 228 + 1


#### ct（定时+矩阵大小+连接数）
##### 协程 50 + 1 + 1
时延：
24641、24123、22815、23777、24148、23420、24915、24341、24468、23600
吞吐量：
86、86、85、86、85、85、85、85、86、86

##### 协程 50 + 1 + 2
时延：
27106、28477、27910、28182、26943、28686、28969、27145、27686、27632
吞吐量：
170、169、170、170、171、170、169、172、170、170

##### 协程 50 + 1 + 4
时延：
33941、35039、34685、35043、36585、34572、34841、37163、34500、34715
吞吐量：
335、336、337、334、333、334、336、335、336、336

##### 协程 50 + 1 + 8
时延：
48249、46923、46902、47196、47112、45288、47291、48024、49785、46474
吞吐量：
640、649、642、636、641、637、646、643、644、642

##### 协程 50 + 1 + 16
时延：
38020、34548、33486、34508、37053、39828、38628、35657、34269、33891
吞吐量：
727、678、696、651、679、709、695、709、655、668

##### 协程 50 + 1 + 32
时延：

吞吐量：



#### ctt（定时+矩阵大小+连接数）
##### 线程 50 + 1 + 1
时延：
43223、40383、41263、39785、42088、41791、41902、44263、44261、45602
吞吐量：
83、83、83、84、84、83、83、83、83、83

##### 线程 50 + 1 + 2
时延：
59070、54927、65116、55354、65931、56760、61855、57679、57195、55164
吞吐量：
165、168、167、166、165、166、166、166、166、166

##### 线程 50 + 1 + 4
时延：
91222、91179、89252、91390、94406、93653、88385、86451、94174、100959
吞吐量：
327、324、330、330、326、327、326、328、325、329

##### 线程 50 + 1 + 8
时延：
153710、168245、173206、168756、165058、167181、132164、132670、160420、162358、161100、166144、172128、175943、192927、120246、158826、166450、140287、127561
吞吐量：
631、634、634、635、626、637、124、139、226、583、371、630、634、634、624、145、269、565、169、157

##### 线程 50 + 1 + 16
时延：
202245、278146、259410、340982、185576、247840、183636、371988、390729、141439、324150、202980、136073、45126、58382、376018、325891、360181、455107、201035
吞吐量：
190、735、449、456、264、220、231、1151、742、107、612、238、60、10、9、1180、501、522、866、207

##### 线程 50 + 1 + 32
时延：

吞吐量：


#### cwpt（定时+矩阵大小+服务器线程）
##### 100 + 40 + 2





























#### apmr
##### 4+128+3876：
2729、2564、2595、2507、2665、2745、2764、2693、2726、2592

#### connect_test
##### 协程 50 + 1 + 20：
28709、28713、28219、28250、28081、28679、29071、27778、29672、29625

##### 协程 50 + 2 + 20：
32835、33062、33291、31886、33922、32637、33675、32905、32388、32642

##### 协程 50 + 4 + 20：
43172、42818、46263、41324、41448、45223、42021、44713、42786、43299

##### 协程 50 + 8 + 20：
62008、64926、73583、51020、78256、52302、63805、68604、70165、59937

##### 协程 50 + 16 + 20：
38127、39866、38954、41001、40924、45048、40691、43185、42311、39012

##### 协程 50 + 32 + 20：
57698、59243、57664、58667、61436、85798、55573、55929、54140、61677

##### 协程 50 + 64 + 20：
120884、134359、112551、


#### connect_thread_test（定时+连接数+矩阵大小）
##### 线程 50 + 1 + 1

#### 串口输出问题，输出不了 total time
- 可能是代码存在的 bug，在 qemu 中体现不出来