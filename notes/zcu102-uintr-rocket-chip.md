# 复现步骤

已有条件：从 sd 卡上启动 linux-xlnx，假设已经安装好了 vivado、scala

1. 将电源接上，旁边的开关拨动之后，等待板子上的灯全部亮绿灯
2. 两个口，一个口为串口，一个为 jtag，接上串口后，可以直接从串口中读取到 arm 侧的 linux 启动信息
3. 从 `https://github.com/U-interrupt/uintr-rocket-chip` 将 `uintr-rocket-chip` 拉取到本地，并进入 `uintr-rocket-chip`
   1. 进入 `zcu102` 子目录，依次执行 `make init`、`make build`、`make checkout`（可能会遇到错误，需要修改 `digilent-vivado-scripts` 目录下的 `config.ini` 中的 vivado 版本和安装目录 ）
4. 从 vivado 中打开 `uintr-rocket-chip/zcu102/proj` 中的项目文件
5. 开始仿真、综合（需要许可证）
6. 进入到 uintr-rocket-chip 中执行 `./reset.sh`，rocket 上的启动的灯闪烁了，但串口没有看到输出
7. PL 侧的串口是第三个，修改串口后，可以看到 PL 侧的 linux 软核已经运行了 Linux，但由于设备树的原因会卡住，后续关于 uintr 的相关信息需要咨询 TKF
