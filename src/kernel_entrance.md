# 内核入口

我们知道，当内核被引导时，我们唯一可以确定的通用寄存器是 pc，包括 sp 在内的其他可写通用寄存器是没有意义的。也就是说，我们甚至没有函数栈。因此，在执行到内核的第一条指令时，我们无法使用任何常规的高级语言，必须使用机器码或者汇编。并利用这些代码设置函数栈。

此外，由于 qemu 会从 0x80000000 开始执行，所以我们还必须将入口点的代码放到单独的一个段里面，并在 linker script 中保证这个段的地址从 0x80000000 开始。

## 编写内核入口的代码

我们可以先编写一个简单的内核入口代码，然后再将其放到一个单独的段中。这个代码的功能很简单，就是设置函数栈，然后调用高级语言的函数（我们这里以 `rust_main` 命名）。

```asm
    .section .text.entry
    .globl _start
_start:
    la sp, boot_stack_top
    call rust_main

    .section .bss.stack
    .globl boot_stack_lower_bound
boot_stack_lower_bound:
    .space 4096 * 16
    .globl boot_stack_top
boot_stack_top:
```

由于 RISCV 中栈是向下生长的，所以我们将栈的起始地址设置为 `boot_stack_top`。

## 嵌入代码到内核

利用 `core::arch::global_asm` 宏，我们可以将上面的汇编代码嵌入到内核中。具体来说，我们需要在一个 rust 源文件中加入如下代码：（假设上述代码保存在 `entry.asm` 文件中）

```rust
use core::arch::global_asm;

global_asm!(include_str!("entry.asm"));
```
