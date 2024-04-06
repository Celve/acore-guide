# 应用地址空间

## Overview

这里对应用地址空间的布局进行简单介绍：

- `.text`：这个区域包含应用程序的代码。它是只读的，以防止程序意外更改其代码，且它是可执行的。
- `.rodata`：即 Read Only Data。顾名思义，这部分包含只读的数据，如代码中使用的常数。这部分显然是不可执行的。
- `.data`：这个段通常包含初始化的全局和静态变量。可读且可写，但不可执行。
- `.bss`：这个区域包含未初始化的全局和静态变量。这些是程序运行时将被填充的内存区域，但在程序开始时并不需要保存有意义的数据。因此，这个段通常被初始化为零。可读且可写，但不可执行。
- `stack`：“程序栈”或“调用栈”用来保存局部变量和函数调用信息。它向更低的内存地址增长。可读且可写，但不可执行。
- `trap context`：这部分用于在处理中断或者陷阱（如系统调用）时保存 CPU 的状态。当陷阱发生时，内核保存当前上下文（寄存器，指令指针等），做完其工作后恢复 trap context。
- `trampoline`：这是一小段代码，用于在执行系统调用或处理中断时页表的切换以及上下文的切换。
- `guard page`：guard page 用于在溢出或越界访问时触发异常。

以下展示了 rCore 中的应用地址空间布局：

## Layout

```plain
+------------------------+ <- end of address space (2^64 Byte)
|       trampoline       | 
+------------------------+
|      trap context      | 
+------------------------+
|          ....          | <- empty
+------------------------+ 
|         stack          |
+------------------------+
|       guard page       |
+------------------------+
|         .bss           |
+------------------------+
|         .data          |
+------------------------+
|        .rodata         |
+------------------------+
|         .text          |
+------------------------+
|       unallocated      |
+------------------------+ <- start of address space
```

rCore 中的 user stack 是定长的，并不是很优秀的设计。如果可能可以删去 guard page，在 page fault 之后再分配，这样可以节省内存。

## Load ELF

我们现在对于应用程序的加载是通过 ELF 文件来进行的。我们需要解析 ELF 文件，将其加载到内存中。我们可以借用 `xmas_elf` crate 来加载 ELF 文件。利用 `xmas_elf` 来读取不同的 map areas，并相应的配置 page table，大致如下（节选自 rCore）：

```rust
let elf = xmas_elf::ElfFile::new(elf_data).unwrap();
let elf_header = elf.header;
let magic = elf_header.pt1.magic;
assert_eq!(magic, [0x7f, 0x45, 0x4c, 0x46], "invalid elf!");
let ph_count = elf_header.pt2.ph_count();
for i in 0..ph_count {
    let ph = elf.program_header(i).unwrap();
    if ph.get_type().unwrap() == xmas_elf::program::Type::Load {
        let start_va: VirtAddr = (ph.virtual_addr() as usize).into();
        let end_va: VirtAddr = ((ph.virtual_addr() + ph.mem_size()) as usize).into();
        let mut map_perm = MapPermission::U;
        let ph_flags = ph.flags();
        if ph_flags.is_read() { map_perm |= MapPermission::R; }
        if ph_flags.is_write() { map_perm |= MapPermission::W; }
        if ph_flags.is_execute() { map_perm |= MapPermission::X; }
        todo!("map the segment to memory in page table");
    }
}
```
