# Trampoline

关于 trampoline，我们需要写的内容包括：[编写 trampoline 的代码](#编写-trampoline-的代码)、[在内核和用户态的地址空间中设置 trampoline](#在内核和用户态的地址空间设置-trampoline)。

## 编写 Trampoline 的代码

在进入 trampoline 时，我们不能破坏除了 pc 以外的所有寄存器，因此代码必须是由汇编编写（而且不能内联汇编）。而且，由于 trampoline 是以页面为单位的，所以 trampoline 的起始位置必须按照页面对齐。

### 设置起始位置对齐

我们需要为 trampoline 单独设置一个段，并在 linker script 中设置按页对齐。比如 trampoline 在 `.trampoline` 段，那么我们需要在 linker script 中修改为：（注意：如果你的段名称是 `.text.` 开头的，请把 trampoline 段放到 `*(.text .text.*)` 前面，并把进入内核的第一行代码放到 trampoline 前面，以免进入错误的内核起始点。

```linker-script

...

    .text : {
        *(.text .text.*)
        . = ALIGN(0x1000);
        _trampoline = .;
        *(.trampoline)
        . = ALIGN(0x1000);
        ASSERT(. - _trampoline == 0x1000, "error: trampoline size is not a page size");
    }

...

```

`_trampoline` 和 `ASSERT` 是保证了 trampoline 会正好占用一个页，以免在内核设置地址时发生错误。

在 trampoline 的汇编中，我们需要将 trampoline 设置为之前指定的段：

```asm
.section .trampoline

... # Your code here
```

### 编写 Trampoline

Trampoline 中一般有两个函数，一个是从用户态进入到内核态的 trap vector，另一个是从内核态进入到用户态的 trap return 函数。这两个函数虽然在用户态和内核态都可见，但是实际使用时，CPU 都已经处于 S mode。

此外，为了方便计算函数偏移量（因为 trampoline 不在内核 linker script 指定的地址，而是在一个用户态和内核态都不会用到的地址），我们还可以加一个 trampoline 起始地址。当然，我们也可以使用 trampoline 中的第一个函数，不过这看起来不太优雅。

从用户态到内核态的函数（这里我们将这个函数成为 `user_trap`）需要做这几件事情：

1. 将寄存器 x1-x31 记录到 trap context 中（提示：你可以[利用 scratch 储存临时变量](asm_coding_tips.md#利用-scratch-储存临时变量)）；
2. 将内核的 sp 寄存器设置为 trap context 中储存的内核 sp 地址；
3. 将内核处理 trap 的函数地址暂存到某个寄存器中；
4. 切换到内核页表（提示：[请这样切换页表](asm_coding_tips.md#切换页表)）；
5. 跳转到内核处理 trap 的函数。

从内核态到用户态的函数（这里我们将这个函数称为 `user_return`）需要做这几件事情（假设 `stvec`、`sstatus`、`sepc` 已经设置妥当，用户页表可以作为入参传入此函数，亦可储存于 trap context 中）：

1. 切换到用户页表（提示：[请这样切换页表](asm_coding_tips.md#切换页表)）；
2. 将 trap context 中储存的用户寄存器 x1-x31 恢复；
3. `sret` 返回用户态。

## 在内核和用户态的地址空间设置 Trampoline

通常情况下，我们会将 trampoline 放在内核地址空间的最高位置，也就是 \\(2^{39}-4096\\) 处。我们需要在内核页表和所有的用户页表中将这个地址映射到 trampoline 实际的地址。
