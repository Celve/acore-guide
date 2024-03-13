# Memory Allocator

## 静态分配和动态分配

## Rust 中的 `GlobalAlloc`

Trait `GlobalAlloc` 的定义可以在[这里](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html)找到。根据[这篇文章](https://os.phil-opp.com/heap-allocation/#the-globalalloc-trait)， `GlobalAlloc` 定义了 `alloc` 和 `dealloc` 两个方法，分别用于分配和释放内存。你写的 heap allocator 需要实现 `GlobalAlloc` 这个 trait，且具体有以下要求：

1. `alloc`：返回一个指向分配的内存的一个字节的指针，如果分配失败则返回 `null`。
2. `dealloc`：释放之前分配的内存。**要求这块内存之前是通过 `alloc` 分配的**。

## Rust 中的堆数据结构

## Buddy Allocator
