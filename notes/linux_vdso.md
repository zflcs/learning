### Linux vDSO 机制学习笔记

在 `arch/xxx/kernel/vdso.c` 中

在 RISC-V 架构的 Linux 系统中，vDSO（virtual Dynamic Shared Object）的启动和初始化过程是系统启动和用户程序执行中的重要环节。以下是该过程的详细描述：

1. **vDSO 的构建**：vDSO 是在内核编译过程中构建的。它通过 `make kernel arch/riscv/kernel/vdso/*.o V=1` 命令生成 `.o` 文件，然后通过链接脚本 `vdso.lds.S` 生成 `vdso.so.dbg` 共享库文件。这个共享库包含了 vDSO 提供的所有系统调用的二进制指令。

2. **vDSO 集成到内核**：在内核源码中，`vdso.S` 文件通过 `.incbin` 指令将 `vdso.so` 共享库包含进来，并确保内存页对齐。然后，使用 `ar` 命令将 `vdso.o` 打包到 `built-in.a` 文件中，最终被打包进内核中。

3. **内核启动时初始化**：内核启动时，会初始化 `vdso_info` 内核对象，该对象包含了 vDSO 代码和数据的地址、虚拟内存映射结构等信息。`vdso_info` 的初始化涉及到设置 vDSO 代码的起始和结束地址，以及数据部分的虚拟内存映射。

4. **用户进程启动时初始化**：当用户进程启动时，会进行以下步骤来初始化 vDSO：
   - 在内核态执行 `execve` 系统调用，将 vDSO 代码和数据映射到用户内存，并将代码地址记录在用户栈内存中。
   - 在用户态执行 dynamic linker，找到 vDSO 代码地址并加载，初始化 vDSO 函数的地址。这涉及到 `setup_vdso` 和 `setup_vdso_pointers` 函数，它们负责设置 vDSO 的数据结构和函数指针。
   - 对于静态链接的程序，在用户态执行 libc init 时，会从辅助向量中找到 vDSO 地址并初始化对应的函数指针。

5. **辅助向量 (Auxiliary Vector)**：在用户进程的辅助向量中，`AT_SYSINFO_EHDR` 项指向 vDSO 代码部分的起始地址。这是通过 `create_elf_tables` 函数设置的，该函数在用户栈上添加辅助向量信息。

6. **vDSO 函数的使用**：一旦 vDSO 初始化完成，用户程序就可以通过 glibc 中的函数（如 `gettimeofday`）来访问 vDSO 提供的服务。这些函数会通过 vDSO 的函数指针调用 vDSO 中的相应代码。

7. **vDSO 数据部分的读写**：用户程序可以通过 `_vdso_data` 对象访问 vDSO 数据部分，而内核可以通过 `vdso_data` 对象访问相同的数据。这允许用户程序高效地读取系统时间等信息。

8. **vDSO 的地址重定位**：操作系统负责在加载共享库到内存后进行地址重定位，确保 vDSO 的代码和数据部分正确映射到用户空间。

通过上述过程，RISC-V 架构的 Linux 系统能够高效地启动和初始化 vDSO，从而提高系统调用的性能并减少用户程序和内核之间的上下文切换开销。