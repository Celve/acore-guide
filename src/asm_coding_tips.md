# RISCV 汇编编写贴士

## 切换页表

在切换页表时，为了防止出现不一致的情况，我们需要先清空 TLB（这个过程保证了旧的页表的数据是正确的），然后切换页表，之后再清空 TLB。

在 RISCV 中，我们可以通过 `sfence.vma` 指令来清空 TLB。

比如在切换页表时，我们可以这样做：（假设 t0 是新的 satp）

```asm
    # clear TLB
    sfence.vma

    # switch page table
    csrw satp, t0

    # clear TLB
    sfence.vma
```

## 利用 scratch 储存临时变量

在编写一些代码时（尤其是 trap vectors），我们需要使用除了通用寄存器之外的寄存器储存临时变量。比如将所有的通用寄存器保存到特定内存地址时，我们还需要寄存器储存内存地址。我们可以利用 RISCV 中的 `mscratch`/`sscratch` 寄存器来储存这些临时变量。

通常情况下，我们会先将一个寄存器的值储存在 scratch 上，然后完成操作后，再还原这个寄存器的值。

比如当我们需要将所有通用寄存器储存到 `TRAP_CONTEXT` 开始的地址时，我们可以：

```asm
    # save user a0 in sscratch so a0 can be used to get at TRAP_CONTEXT.
    csrw sscratch, a0
    li a0, TRAP_CONTEXT

    # save the user registers except a0 in TRAP_CONTEXT
    ...
    ...

    # save previous a0 to TRAP_CONTEXT
    csrr t0, sscratch
    sd t0, 112(a0)
```
