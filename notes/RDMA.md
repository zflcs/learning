优势：
软件协议栈卸载到网卡
DMA，直接访问内存

单向原语：read、write、atomic
双边原语：send、recv（双方参与，send、recv 存在顺序）
RC、UC、UD

linux rdma 编程：librdmacm + libibverbs

RC 需要使用其他方式传递 QP
UD 则不需要

CPU 与 rdma 网卡之间的交互：
CPU 主动写：
NIC DMA 主动拉取：
