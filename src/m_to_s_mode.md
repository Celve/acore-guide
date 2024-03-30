# 从 M mode 到 S mode

我们现在的内核还没有切换过特权级，也就是说我们现在正在 M mode 下运行。我们需要切换到 S mode 下运行我们的内核。

为了能让我们的内核正确地切换内核，我们大致需要做以下几件事情：

1. 设置 `mstatus` 寄存器及 `mepc` 寄存器；
2. 暂时禁用页表；
3. 启用中断（为后面时间片调度做准备）；
4. 配置 Physical Memory Protection，让内核能在 S mode 下访问所有物理内存；
5. 启用时钟中断；
6. 调用 mret 指令，切换到 S mode。

## 设置 mstatus 寄存器及 mepc 寄存器

我们需要将 `mstatus` 寄存器的 `MPP` 字段设置为 1，这样当我们执行 `mret` 指令时，我们就会切换到 S mode。然后我们需要将 `mepc` 寄存器设置为内核 S mode 的入口地址。

## 暂时禁用页表

将 `satp` 寄存器设置为 0 即可。

## 启用中断

首先，将 `medeleg` 和 `mideleg` 寄存器设置为 `0xffff`。

将 `sie` 寄存器的 `SEIE`、`STIE` 和 `SSIE` 位设置为 1，这样当我们切换到 S mode 时，中断会被启用。

## 配置 Physical Memory Protection

我们需要将 `pmpaddr0` 寄存器设置为 `0x3fffffffffffff`，并将 `pmpcfg0` 寄存器设置为 `0xf`。

## 启用时钟中断

我们需要将 `mtimecmp` 寄存器设置为一个合适的值，这样当时钟中断到来时，CPU 会跳转到内核的时钟中断处理函数。

间隔的时间一般是 `1000000`，这样大概 0.1 秒会产生一次时钟中断。

由于 RISCV 中 `mtime` 和 `mtimecmp` 寄存器都是 Memory-Mapped 地址的，所以我们只需要操作内存地址即可。

根据 qemu 的 `hw/riscv/virt.c` 文件，`mtime` 和 `mtimecmp` 寄存器的地址分别是 `0x0200bff8` 和 `0x02004000 + 8 * hartid`。

在初始化时，我们需要先设置何时的 `mtimecmp` 寄存器（用 `mtime` 加上时间间隔）。里面的一些参数可以通过 mscratch 寄存器来传递。

接着，将 `mtvec` 寄存器设置为时钟中断处理函数的地址。注意，为了放置上下文被破坏，时钟中断处理函数入口一定是汇编。

然后，通过设置 `mstatus` 寄存器的 `MIE` 位来启用 M mode 中断。

最后，通过设置 `mip` 寄存器的 `MTIP` 位来使时钟中断生效。

## 调用 mret 指令

最后，我们需要调用 `mret` 指令，切换到 S mode。
