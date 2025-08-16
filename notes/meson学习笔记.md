# Meson 学习笔记

## 编译 Meson 项目

```shell
cd /path/to/source/root
meson setup builddir && cd builddir
meson compile
meson test
```

Meson不允许在源代码树内构建源代码。所有构建工件都存储在构建目录中。这允许同时拥有具有不同配置的多个构建树。避免了编译产物对版本管理工具的影响。

在编译一次后，如果代码修改了，只需要运行 `meson compile`。如果需要构建优化的二进制文件，只需在运行 Meson 时使用参数 `--buildtype=debugoptimized`。（建议一个目录是未优化的，另一个为优化的，各自编译的方式是进入对应的 builddir 执行 `meson compile`）

`-g` 添加 debug 信息，`-Wall` 使能编译器警告。

## Using Meson as a distro packager

```shell
cd /path/to/source/root
meson --prefix /usr --buildtype=plain builddir -Dc_args=... -Dcpp_args=... -Dc_link_args=... -Dcpp_link_args=...
meson compile -C builddir
meson test -C builddir
DESTDIR=/path/to/staging/root meson install -C builddir
```

`--buildtype=plain` 告诉 Meson 不要向命令行添加自己的标志。这给了打包者对所使用标志的完全控制。

`DESTDIR` 变量是以环境变量的形式传递，而不是作为 `meson install `的参数。



## 添加额外的依赖

创建一个叫 `subprojects` 的子目录，之后可以通过 `meson wrap install XXX`，会在 `subprojects` 目录下新增一个 `XXX.wrap` 的文件，之后再在 `meson.build` 中添加对应的依赖。



