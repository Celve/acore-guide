# 特权寄存器指南

本文档将介绍如何方便地修改和读取特权寄存器，以及修改和读取特权寄存器的特定字段。

rCore 的 [riscv 库][rcore-riscv-lib]提供了较为全面的特权寄存器支持。利用这个库，我们可以更方便地操作特权寄存器。

[rcore-riscv-lib]: https://github.com/rcore-os/riscv

## 导入 riscv 库

首先，在 `Cargo.toml` 的 `[dependencies]` 中加一行

```plain
riscv = { git = "https://github.com/rcore-os/riscv", features = ["inline-asm"] }
```

然后在需要使用 riscv 库的文件中 `use riscv::register::<reg>;` 即可使用对应的寄存器。如果使用的寄存器较多，也可以 `use riscv::register::*;` 导入所有的寄存器。

## 修改特权寄存器

我们可以使用寄存器的 `write` 方法来写入特权寄存器。

例如我们需要将 `mepc` 设置为 `main` 函数（这个操作在从 m mode 到 s mode 中很重要，函数名可以不同），我们可以（需要在函数中调用 `write`）

```rust
use riscv::register::mepc;

...

    mepc::write(rust_main as usize);
```

## 读取特权寄存器

我们可以使用寄存器的 `read` 方法来读取特权寄存器。

例如我们需要读取 `sstatus`，我们可以

```rust
use riscv::register::sstatus;

...

   let sstatus = sstatus::read();
```

## 修改寄存器的特定字段

我们可以使用寄存器的 `set_<name>` 方法来修改特定字段。

例如我们需要修改 `mstatus` 的 `MPP` 位，我们可以

```rust
use riscv::register::mstatus;

...

    mstatus::set_mpp(riscv::register::mstatus::MPP::Supervisor);
```

## 读取寄存器的特定字段

我们可以使用寄存器的 `<name>` 方法来读取特定字段。

例如我们需要读取 `mstatus` 的 `MPP` 位，我们可以

```rust
use riscv::register::mstatus;

...

    let mpp = mstatus::mpp();
```

