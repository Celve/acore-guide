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

Trampoline 中一般有两个函数，一个是从用户态进入到内核态的 trap vector，另一个是从内核态进入到用户态的 trap return 函数。

此外，为了方便计算函数偏移量（因为 trampoline 不在内核 linker script 指定的地址，而是在一个用户态和内核态都不会用到的地址），我们还可以加一个 trampoline 起始地址。当然，我们也可以使用 trampoline 中的第一个函数，不过这看起来不太优雅。

## 在内核和用户态的地址空间设置 Trampoline
