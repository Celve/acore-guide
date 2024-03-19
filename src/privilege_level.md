# 特权级机制

*我们这里只会介绍原理和从 M mode 转到 S mode 的方法。*

我们已经在[操作系统简介](os_intro.md#用户态内核态)中介绍了特权级及其作用。我们知道，特权级机制可以保证应用的运行不会影响到内核或者其他的应用。

整个特权级机制是通过硬件加上指令集支持实现的。CPU 有一些特殊的寄存器，叫做 CSR (Control and Status Register)，用于控制和记录特权相关的状态。

我们将从高特权态到低特权态的操作称为 `eret` (environment return)，将低特权态到高特权态的操作成为 `ecall`。

## 特权指令

RISCV 大致提供这些特权指令：

| 指令 | 描述 |
|:---:| --- |
| `ecall` | 从低特权态进入高特权态 |
| `ebreak` | 用于调试 |
| `mret` | 从机器态到低特权态，在特权态不正确的情况下执行会产生非法指令异常 |
| `sret` | 从监管态到低特权态，在特权态不正确的情况下执行会产生非法指令异常 |
| `wfi` | 等待中断，在 U mode 下执行会产生非法指令异常 |
| `sfence.vma` | 刷新 TLB，在 U mode 下执行会产生非法指令异常 |
| `csrr` | 读取 CSR（到通用寄存器） |
| `csrw` | 写入 CSR（从通用寄存器） |
| `csrrw` | 读取 CSR 并写入 CSR，这个过程是原子的 |

## 一些特权寄存器或寄存器位的命名

我们会在 RISCV 的手册中看到一些特权寄存器或寄存器位的命名，但是由于体系结构喜欢用简称来称呼，而简称对于不熟悉的人来说不是很友好，这里我们介绍一些比较常见的命名。

- `mstatus`/`sstatus` (Machine/Supervisor Status)：记录特权级的状态。
- `mpp`/`spp` (Machine/Supervisor Previous Privilege)：记录上一次的特权级。
- `mpie`/`spie` (Machine/Supervisor Previous Interrupt Enable)：记录上一次的中断使能状态，这是由于当发生中断时，中断会被禁用，以免在处理中断时又发生中断。
- `mepc`/`sepc` (Machine/Supervisor Exception Program Counter)：记录上次异常的地址。
- `mcause`/`scause` (Machine/Supervisor Cause)：记录异常的原因。
- `mtval`/`stval` (Machine/Supervisor Trap Value)：记录异常的附加信息。
- `medeleg`/`mideleg` (Machine Exception/Interrupt Delegation)：处理异常/中断的特权态。
- `mie`/`sie` (Machine/Supervisor Interrupt Enable)：中断启用。
- `mip`/`sip` (Machine/Supervisor Interrupt Pending)：有中断等待。
- `mscratch`/`sscratch` (Machine/Supervisor Scratch)：用于保存临时数据，可以利用如 `csrrw a0, mscratch, a0` 来交换 mscratch 和通用寄存器的值。
- `satp` (supervisor address translation and protection register)：用于设置虚拟内存映射规则。
