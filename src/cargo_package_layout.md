# Rust 项目的 Cargo 包布局教程

当您使用 Rust 进行项目开发时，有效的文件结构组织对于项目管理和可维护性至关重要。Cargo —— Rust 的包管理器和构建工具，定义了一套文件放置的约定来简化了解新 Cargo 包的过程。以下是如何根据这些约定来组织您的 Rust 项目的教程。

## 基本文件结构

Cargo 包的典型目录结构如下所示：

```****
.
├── Cargo.lock
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── main.rs
│   └── bin/
│       ├── named-executable.rs
│       ├── another-executable.rs
│       └── multi-file-executable/
│           ├── main.rs
│           └── some_module.rs
├── benches/
│   ├── large-input.rs
│   └── multi-file-bench/
│       ├── main.rs
│       └── bench_module.rs
├── examples/
│   ├── simple.rs
│   └── multi-file-example/
│       ├── main.rs
│       └── ex_module.rs
└── tests/
    ├── some-integration-tests.rs
    └── multi-file-test/
        ├── main.rs
        └── test_module.rs
```

- **Cargo.toml 和 Cargo.lock**：这两个文件存储在包的根目录（_package root_），包含项目的基本信息和一些 dependencies。
- **src 目录**：源代码放在这里，其中 `src/lib.rs` 是默认的库文件，`src/main.rs` 是默认的执行文件。
  - 其他执行文件可以放在 `src/bin/` 目录下。
- **benches 目录**：用于存放基准测试文件，以测试代码性能。
- **examples 目录**：里面放的是示例代码，通常展示如何使用库。
- **tests 目录**：用于集成测试文件，检查不同模块是否正常协作。

## Cargo Build

### How To Use

当你运行 `cargo build` 命令时，Cargo 会自动寻找项目中所有的可编译资源，包括 `lib.rs`、`main.rs` 以及 `bin/` 目录下的每个 `.rs` 文件。每个源码文件都会被编译成独立的二进制可执行文件。

- **编译整个项目**：简单运行 `cargo build` 会构建项目中的所有可执行文件和库。
- **编译特定的二进制文件**：如果你只想编译 `bin/` 目录中的特定文件，可以使用 `cargo build --bin <name>`。

### What Happened?

考虑一次 `cargo build` 的过程：

1. **解析依赖**：Cargo 首先读取 `Cargo.toml` 文件，解析项目依赖。
2. **编译库**：如果存在 `lib.rs`，Cargo 会首先编译库。
3. **编译二进制应用**：随后，Cargo 编译 `src/main.rs`（如果存在）和 `bin/` 目录下的所有文件，为每个文件生成一个可执行文件。
4. **放置可执行文件**：编译成功后，可执行文件将被放置在 `target/debug/` 目录下，文件名与 `bin/` 目录中的源文件名相匹配。
