
# 复现 AnTiQ

### 20240116

1. fork 到 ATS-INTC 仓库下
2. 在 fpga 子目录下的 makefile 里有使用 vivdo 进行测试的脚本，项目创建成功，但是测试失败了。
   1. 许可证
   2. TIME_WIDTH 参数未定义
   3. 看了一下生成的项目，推测可能是因为顶层的 test_bench 模块设置错误
3. 手动创建 vivdo 项目，将 src 目录下的 tb_pq_common 设置为顶层模块
   1. 但里面的随机生成函数用法有错误，导致运行不成功
4. 给原仓库提 issue
5. 把 src 里面的 TIME_WIDTH 改成 DATA_WIDTH，TW 改成 DW，把 fpga/rtl 下的 pq_fpga_top.sv 里模块参数 push_id_0 改成 push_id_i，对应的 output 改成 input，成功跑起来了测试
   1. 测试的结果主要是 LUT 使用率等
6. 准备在 AXU15EG 上跑起来

