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



## 编译类型 `--buildtype`

1. plain：不使用额外的构建参数
2. debug：生成调试信息，但结果未优化
3. debugoptmized：生成调试信息并优化代码（`-g -O2`）
4. release：全面优化，无调试信息



## 编译后端

默认使用 ninja，可以通过 `--backend=xx` 来设置其他后端。

编译可以直接使用 `ninja -C builddir`



## 测试

`meson test -C builddir` 或 `ninja -C builddir test`

不强制使用任何特定的测试框架，可以自由使用 GTest、Boost Test、Check 等。



## 安装构建的软件

`meson install -C builddir`，默认安装到 `/usr/local`，可以通过 `--prefix /your/prefix` 来安装到指定的目录，或者使用 `DESTDIR` 环境变量。

也可以使用 `ninja -C builddir install`



## 错误

0 表示成功，1 表示 meson.build 文件出现问题，2 表示内部出错。



## 工作流程

setup $\rightarrow$ compile $\rightarrow$ install



## 帮助

1. `meson configure -h` 查看配置的帮助
2. `meson compile -h` 查看编译的帮助



## 发布

`meson dist` 从当前源码树发布存档



## 初始化

`meson init` 基于一组模板创建一组基本的构建文件，可以使用 `-h` 来查看帮助。

1. -n 表示项目名字，name 默认为当前目录的名字
2. -e 可执行文件名字，默认为项目名
3. -d 依赖，使用逗号分隔
4. -l 使用的语言，默认为自动检索源文件的语言
5. -b 构建目录
6. --type 项目类型，可执行文件或库



## env2mfile

`meson env2mfile`创建本地和交叉编译的文件，使用 `-h` 查看帮助。

1. --gccsuffix：gcc 版本后缀
2. -o：输出文件
3. --cross：生成交叉编译文件
4. --native：生成本地编译文件
5. --use-for-build：使用 `_FOR_BUILD` 环境变量
6. --system： 定义交叉编译的系统
7. --subsystem：定义交叉编译的子系统
8. --kernel：定义交叉编译的内核
9. --cpu：定义交叉编译的 CPU
10. --cpu-family：定义交叉编译的 CPU 家族
11. --endian：定义交叉编译的字节序



## introspect

`meson introspec` 可以打印出一些信息



## reprotest

`meson reprotest`可编译两次并检查最终结果是否相同，必须要再测试的项目的源根目录中运行。



## rewrite

`meson rewrite` 修改 meson 项目



## setup

`meson setup --clearcache --reconfigure <builddir>` 可以在单个命令中清除缓存并重新配置。



## subprojects

`meson subprojects` 管理子项目，`-h` 查看帮助



## wrap

`meson wrap` 管理 WrapDB 依赖项的实用程序



## devenv

`meson devenv` 用于运行命令，或者打开交互式的 shell



## subdir

在 `meson.build` 中使用 `subdir()`，会执行给定的子目录中的 `meson.build` 中的内容，并且所有的状态（变量）都会传入子目录



## 交叉编译

```meson
# aarch64.ini
[constants]
arch = 'aarch64-linux-gnu'
```

```meson
# cross.ini
[binaries]
c = arch + '-gcc'
cpp = arch + '-g++'
strip = arch + '-strip'
pkg-config = arch + '-pkg-config'
...
```

`meson setup --cross-file aarch64.ini --cross-file cross.ini builddir`

文件组合发生在 meson 解析变量之前，



