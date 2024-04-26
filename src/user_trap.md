# 处理用户态的 Trap

用户态时如果发生中断或异常，都会进入 `stvec` 所指定的地址。这个地址就是我们前面所说的 trampoline。不过 trampoline 只会完成最基础的工作如保存上下文，切换页表等。完成这些之后，我们可以进入到内核中，使用高级语言完成更多的工作。

Trampoline 的 `user_trap` 最后会跳转到一个内核中的函数，这个函数需要进一步记录一些信息（如 `sepc`，这些信息可以储存在函数栈中），识别中断或异常的原因，并调用相应的处理函数。

## 识别中断或异常的原因

在 S mode 的 trap 中，储存 trap 原因的寄存器为 `scause`。`scause` 的低 63 位是 trap 的原因，最高位为 0 表示异常，为 1 表示中断。其余位表示异常或中断的原因。下表是 RISCV 的手册中所写的 `scause` 在不同情况下的含义。

| Interrupt |   Exception | Description                    |
| --------: | ----------: | ------------------------------ |
|         1 |           0 | *Reserved*                     |
|         1 |           1 | Supervisor software interrupt  |
|         1 |         2-4 | *Reserved*                     |
|         1 |           5 | Supervisor timer interrupt     |
|         1 |         6-8 | *Reserved*                     |
|         1 |           9 | Supervisor external interrupt  |
|         1 |       10-15 | *Reserved*                     |
|         1 | \\(\ge\\)16 | *Designated for platform use*  |
|         0 |           0 | Instruction address misaligned |
|         0 |           1 | Instruction access fault       |
|         0 |           2 | Illegal instruction            |
|         0 |           3 | Breakpoint                     |
|         0 |           4 | Load address misaligned        |
|         0 |           5 | Load access fault              |
|         0 |           6 | Store/AMO address misaligned   |
|         0 |           7 | Store/AMO acess fault          |
|         0 |           8 | Environment call from U-mode   |
|         0 |           9 | Environment call from S-mode   |
|         0 |       10-11 | *Reserved*                     |
|         0 |          12 | Instruction page fault         |
|         0 |          13 | Load page fault                |
|         0 |          14 | *Reserved*                     |
|         0 |          15 | Store/AMO page fault           |
|         0 |       16-23 | *Reserved*                     |
|         0 |       24-31 | *Designated for custom use*    |
|         0 |       32-47 | *Reserved*                     |
|         0 |       48-63 | *Designated for custom use*    |
|         1 | \\(\ge\\)64 | *Reserved*                     |

我们不一定要完全将这所有的错误原因都处理。如 breakpoint 问题，正常情况下是不会遇到的。

在中断的所有 `scause` 中，我们只会遇到 supervisor software interrupt（软中断）和 supervisor external interrupt（外部中断）。软中断通常被用来在时钟中断时发起。外部中断意味着有硬件触发了中断。这个设备可以是 UART，也可以是 VIRTIO 的 IRQ（后面会在 FS 中介绍）。

在异常的所有 `scause` 中，较为常见的是 environment call from U-mode，也就是用户态发起的系统调用。此外，你还可以按需处理各种 page fault，以及其他的错误。

## Debug

`scause` 寄存器是一个很好的调试信息。你可以在 trap 处理函数中打印这个值，以便更好地了解发生了什么。

当你的程序出现了异常，你可以通过这个值来判断异常的原因。比如，如果你的程序出现了 page fault，你可以通过 `scause` 的值来判断是哪种 page fault，然后进一步调试。
