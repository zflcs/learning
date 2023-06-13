
将文件复制到对应的目录中

在 include/linux 中增加 xilinx_phy.h 头文件

修改了 xilinx_axienet_main.c 中 netif_napi_add 的参数

只用一个核启动不了 uintr-rocket-chip 分支，加了 -smp 2 之后正常（发现是没清除缓存）