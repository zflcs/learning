### rCore 中的网络驱动

从 rCore/kernel/net/mod.rs 文件中看到，使用了 smoltcp 这个依赖库，于是从 cargo.toml 中找到对应的 github 地址

获取 IP 数据报 IPID 的函数 [ipv4](https://github.com/rcore-os/smoltcp/blob/master/src/wire/ipv4.rs#L329)、[ipv6](https://github.com/rcore-os/smoltcp/blob/master/src/wire/ipv6fragment.rs#L93)

设置 IP 数据报 IPID 的函数[ipv4](https://github.com/rcore-os/smoltcp/blob/master/src/wire/ipv4.rs#L448)、[ipv6](https://github.com/rcore-os/smoltcp/blob/master/src/wire/ipv6fragment.rs#L141)

在 smoltcp 中提供了这些字段的接口，但是在 rCore 和 zCore 中并没有进行处理，在发送数据报时，ID 字段和 flag 字段默认位为 0

### redox 操作系统的网络驱动



