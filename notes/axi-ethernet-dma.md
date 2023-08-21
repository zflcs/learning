### axi-ethermet

阅读手册需要注意里面的 note 等信息

is 寄存器中的 mgt_rdy 只有在使用 sgmii 和 1000BaseX 时才会为 1，因此在 reset 时会先 return NO_MGTRDY 是正常

mac_irq 在 mdio_ready 或  interrupt_ptp_rx 或  interrupt_ptp_tx 或  interrupt_ptp_timer 时为 1，后三个中断只有在 使能 AVB 的情况下才会出现

