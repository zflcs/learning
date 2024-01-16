# QEMU SIFIVE PLIC 学习笔记

### 一些基本概念

AIA（Advanced Interrupt Architecture）：由 IMSIC（Incoming MSI Controller） 和 APLIC（Advanced Platform-Level Interrupt） 组成。

IMSIC：接收和处理 MSI 中断

APLIC：接收和处理线中断

MSI 中断（Message Signaled Interrupt）：PCI 设备将 Message Data 写入到 Message Address 地址，从而触发 CPU 中断。

### qemu virt 平台创建 sifive_plic 流程：

1. 在 `virt_machine_init` 函数中，`aia_type` 为 `VIRT_AIA_TYPE_NONE`，调用 `virt_create_plic` 函数（暂时没找到在哪里设置 `aia_type`）。

2. 在 `virt_create_plic` 函数中，先生成了 `plic_hart_config` 为 `MS`，再调用 `sifive_plic_create` 函数。

3. `sifive_plic_create` 
   1. 先使用 `qdev_prop_set_string` 初始化了 `SiFivePLICState` 结构体中的成员。
   2. 再使用 `sysbus_mmio_map` 添加 `mmio` 地址映射。
   3. 


```c
/*
 * Create PLIC device.
 */
DeviceState *sifive_plic_create(hwaddr addr, char *hart_config,
    uint32_t num_harts,
    uint32_t hartid_base, uint32_t num_sources,
    uint32_t num_priorities, uint32_t priority_base,
    uint32_t pending_base, uint32_t enable_base,
    uint32_t enable_stride, uint32_t context_base,
    uint32_t context_stride, uint32_t aperture_size)
{
    DeviceState *dev = qdev_new(TYPE_SIFIVE_PLIC);
    int i;
    SiFivePLICState *plic;

    assert(enable_stride == (enable_stride & -enable_stride));
    assert(context_stride == (context_stride & -context_stride));
    qdev_prop_set_string(dev, "hart-config", hart_config);
    qdev_prop_set_uint32(dev, "hartid-base", hartid_base);
    qdev_prop_set_uint32(dev, "num-sources", num_sources);
    qdev_prop_set_uint32(dev, "num-priorities", num_priorities);
    qdev_prop_set_uint32(dev, "priority-base", priority_base);
    qdev_prop_set_uint32(dev, "pending-base", pending_base);
    qdev_prop_set_uint32(dev, "enable-base", enable_base);
    qdev_prop_set_uint32(dev, "enable-stride", enable_stride);
    qdev_prop_set_uint32(dev, "context-base", context_base);
    qdev_prop_set_uint32(dev, "context-stride", context_stride);
    qdev_prop_set_uint32(dev, "aperture-size", aperture_size);
    sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
    sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, addr);

    plic = SIFIVE_PLIC(dev);

    for (i = 0; i < plic->num_addrs; i++) {
        int cpu_num = plic->addr_config[i].hartid;
        CPUState *cpu = qemu_get_cpu(cpu_num);

        if (plic->addr_config[i].mode == PLICMode_M) {
            qdev_connect_gpio_out(dev, cpu_num - hartid_base + num_harts,
                                  qdev_get_gpio_in(DEVICE(cpu), IRQ_M_EXT));
        }
        if (plic->addr_config[i].mode == PLICMode_S) {
            qdev_connect_gpio_out(dev, cpu_num - hartid_base,
                                  qdev_get_gpio_in(DEVICE(cpu), IRQ_S_EXT));
        }
    }

    return dev;
}
```

```c
struct SiFivePLICState {
    /*< private >*/
    SysBusDevice parent_obj;

    /*< public >*/
    MemoryRegion mmio;
    uint32_t num_addrs;
    uint32_t num_harts;
    uint32_t bitfield_words;
    uint32_t num_enables;
    PLICAddr *addr_config;
    uint32_t *source_priority;
    uint32_t *target_priority;
    uint32_t *pending;
    uint32_t *claimed;
    uint32_t *enable;

    /* config */
    char *hart_config;
    uint32_t hartid_base;
    uint32_t num_sources;
    uint32_t num_priorities;
    uint32_t priority_base;
    uint32_t pending_base;
    uint32_t enable_base;
    uint32_t enable_stride;
    uint32_t context_base;
    uint32_t context_stride;
    uint32_t aperture_size;

    qemu_irq *m_external_irqs;
    qemu_irq *s_external_irqs;
};
```



