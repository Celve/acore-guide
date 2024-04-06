# GDB 操作指南

## 基础 GDB 命令及示例

不出意外的话，当你们执行 GDB 时将会看到以下内容：

```plain
GNU gdb (GDB) 13.2
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/g
pl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pc-linux-gnu --target=riscv
64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
--Type <RET> for more, q to quit, c to continue without paging--
cumentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
Reading symbols from target/riscv64gc-unknown-none-elf/release/kernel
...
The target architecture is set to "riscv:rv64".
Remote debugging using localhost:1234
0x0000000000001000 in ?? ()
(gdb) 
```

### Help

所有指令都可以查看 GDB 的帮助文档，只需要在 GDB 中输入 `help command` 即可。例如

```plain
(gdb) help x
```

其显示：

```plain
Examine memory: x/FMT ADDRESS.
ADDRESS is an expression for the memory address to examine.
FMT is a repeat count followed by a format letter and a size letter.
Format letters are o(octal), x(hex), d(decimal), u(unsigned decimal), t(binary), f(float), a(address), i(instruction), c(char), s(string) and z(hex, zero padded on the left).
Size letters are b(byte), h(halfword), w(word), g(giant, 8 bytes).
The specified number of objects of the specified size are printed according to the format.  If a negative number is specified, memory is examined backward from the address.

Defaults for format and size letters are those previously used.
Default count is 1.  Default address is following last thing printed
with this command or "print".
```

### 查看 Assembly 与 Memory

首可以通过 `layout asm` 来查看汇编代码，其会将 GDB 的窗口分为上下两部分，上部分显示汇编代码，下部分是调试窗口。

当然，你也可以通过 examine memory 来查看任意地址的汇编代码，例如：

```plain
(gdb) x/10i 0x1000
```

这个命令会展示地址 `0x1000` 开始的 10 条汇编指令。其中，`10` 代表展示的指令数量，`i` 代表展示的格式。支持的格式可以在 `help x` 中查看。

同样的，你可以利用 `x` 来查看内存，例如：

```plain
(gdb) x/10x 0x8001c0a0
```

这个命令会展示地址 `0x8001c0a0` 开始的 10 个 4 字节的内存单元。

### 设置断点

#### 利用地址设置断点

我们一开始进入 QEMU 的地址是 `0x1000`，我们可以在 `0x80000000` 处设置断点，方便调试，例如：

```plain
b *0x80000000
Breakpoint 1 at 0x80000000: file src/main.rs, line 67.
```

注意，这里的 `b` 是 `break` 的缩写。

`*` 表示后面值是一个 address 而不是一个 symbol，因为我们也可以利用 symbol 来设置断点。

#### 利用符号设置断点

我们可以利用符号来设置断点。我们以 [recore 中的 entry point](https://github.com/Celve/recore/blob/4986f29038c19fc09dde544f86098d732ce34abf/kernel/src/main.rs#L66) 为例：

```plain
b _start
Breakpoint 2 at 0x80000000: file src/main.rs, line 67.
```

便可以在 `_start` 这个函数的入口处设置断点，但需要注意的是，由于 Rust 的函数名会被 mangle，所以我们需要在函数名前加上 `#[no_mangle]` 来防止编译器修改函数名，例如：

```rust
#[no_mangle]
unsafe extern "C" fn _start() {
    todo!()
}

#[no_mangle]
fn foo() {
    todo!()
}

#[no_mangle]
unsafe fn bar() {
    todo!()
}
```

一般来说，只有加上 `#[no_mangle]` 的函数名才能在 GDB 中被识别，可以通过 `info address` 来查看 symbol 的地址，例如：

```plain
(gdb) info address _start
Symbol "_start" is at 0x80000000 in a file compiled without debugging.
```

同理，我们可以在 GDB 中找到 `foo` 和 `bar` 这两个 symbol，并设置断点。

#### 利用文件名和行号设置断点

我们也可以利用文件名和行号来设置断点，假设我们的根目录下有一个文件 `src/main.rs`，我们可以利用 `b src/main.rs:67` 来设置断点于[此处](https://github.com/Celve/recore/blob/4986f29038c19fc09dde544f86098d732ce34abf/kernel/src/main.rs#L67)：

```rust
66 unsafe extern "C" fn _start() {
67     asm!(
68         "la sp, {bootloader_stack}",
69         "li t0, {bootloader_stack_size}",
70         "csrr t1, mhartid",
71         "addi t1, t1, 1",
72         "mul t0, t0, t1",
73         "add sp, sp, t0",
74         "j {rust_start}",
75         bootloader_stack = sym BOOTLOADER_STACK_SPACE,
76         bootloader_stack_size = const BOOTLOADER_STACK_SIZE,
77         rust_start = sym rust_start,
78         options(noreturn),
79     );
80 }
```

只需要：

```plain
(gdb) b src/main.rs:67
Breakpoint 3 at 0x80000000: file src/main.rs, line 67.
```

再举一个例子，例如在[87行](https://github.com/Celve/recore/blob/4986f29038c19fc09dde544f86098d732ce34abf/kernel/src/main.rs#L88)处设置断点（这个函数不需要 `#[no_mangle]`：

```rust
85 unsafe fn rust_start() -> ! {
86     mstatus::set_mpp(riscv::register::mstatus::MPP::Supervisor);
87     mepc::write(rust_main as usize);
88     todo!("...")
89 }
```

```gdb
(gdb) b src/main.rs:88
Breakpoint 4 at 0x8001c0cc: file src/main.rs, line 88.
```

### 逐行执行

利用 `si` 或者 `ni` 逐行执行代码：

```plain
(gdb) si
(gdb) ni
```

当程序停在某个断点时，使用此命令可以一行行地跟踪程序执行流程，包括进入函数体内部。

### 打印寄存器

`p` 可以查看某个寄存器当前的值：

```plain
(gdb) p $pc
```

将 `$pc` 换成你想要查看的寄存器即可。

### 打印变量

`p` 可以打印某个变量的值。譬如说，我想打印[180行处](https://github.com/Celve/recore/blob/4986f29038c19fc09dde544f86098d732ce34abf/kernel/src/main.rs#L180)的 `hart_id`：

```rust
fn init_devices() {
    let hart_id = Processor::hart_id();
    ...
}
```

当执行完 `b src/main.rs:180` 后，我们可以打印 `hart_id` 的值：

```plain
(gdb) print hart_id
```

同时，我们可以利用 `info locals` 来显示所有可供打印的所有局部变量。

### 执行到断点

`c` 或 `continue` 从当前停止点继续执行程序，直到遇到下一个断点或程序结束：

```plain
(gdb) c
```
