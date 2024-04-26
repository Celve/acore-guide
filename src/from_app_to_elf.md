# 从应用到 ELF

## Layout

在 [Cargo Package Layout](cargo_package_layout.md) 中，我们介绍了常见的 Rust 项目布局。类似地，为了编写且编译用户程序，我们可以采取以下布局：

```plain
.
└── user/
    ├── lib.rs
    ├── main.rs
    └── bin/
        ├── hello.rs
        ├── print.rs
        └── ...
```

执行 `cargo build` 之后，我们会在 `user/target/$(TARGET)/$(MODE)` 目录下生成用户程序的 ELF 文件。其中，`$(TARGET)` 是目标平台（理论上只能是 `riscv64gc-unknown-none-elf`），`$(MODE)` 是编译模式（`debug` 或 `release`）。

## Link

Link 中最重要的是我们需要制定 `.text` 的位置，这样我们才能在内核中正确加载用户程序。

## objcopy

使用 `objcopy` 可以将 `cargo build` 生成的 ELF 文件删除所有 ELF header 和符号得到 `.bin` 后缀的纯二进制镜像文件，例如：

```bash
rust-objcopy --binary-architecture=riscv64 --strip-all -O binary target/riscv64gc-unknown-none-elf/debug/user target/riscv64gc-unknown-none-elf/debug/user.bin
```

之后我们需要把 binary 文件加载进内存，并让内核理解并执行之。关于内核读取 ELF 的部分，将在[下一章节](app_address_space.md)中讲解。
