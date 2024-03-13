# Memory Allocator

## 静态分配和动态分配

我们先考虑用户态程序的内存分配问题。内存分配的方式可以分为静态内存分配和动态内存分配

静态内存分配是程序运行前在编译阶段由编译器确定的，不能在运行时进行改变。其通常在堆栈（stack）中分配。

静态内存分配显而易见的问题是不够灵活，无法适应一些需要动态分配内存的情况。一个经典的例子是变长数组，在 Rust 中则是 `std::vec::Vec`。

与静态内存分配不同，动态内存分配的大小和生命周期在编译时并不确定，而是在运行时进行决定和分配。例如 C 中的 `malloc` 和 `free`：

```C
void* malloc (size_t size);
void free (void* ptr);
```

`malloc` 用于分配一块指定大小的内存，`free` 用于释放之前分配的内存。这种动态内存分配的方式更加灵活，但也更加复杂，需要程序员自己管理内存的生命周期。

在 `malloc` 和 `free` 的背后，是用户态的内存分配器，它负责管理内存的分配和释放，当内存不够时使用 `sbrk` 或 `mmap` 向操作系统申请更多的内存。其如何实现可以参考 [Malloc Lab](https://www.cs.cmu.edu/afs/cs/academic/class/15213-f10/www/labs/malloclab-writeup.pdf)。

让我们回到 kernel。我们在 kernel 中也有使用 `Vec`、`Box` 和 `Map` 的需要。但在我们的 kernel 中，由于 `#![no_std]` 的存在，我们无法使用 `std::alloc` 中的内存分配器，自然也就无法使用以上的数据结构。但 Rust 在 `std` 之外提供了 `alloc` crate，我们可以使用这个 crate 来使用其中提供的 `alloc::vec::Vec`，但前提是我们需要自己实现一个内存分配器。

## Rust 中的 `GlobalAlloc`

Trait `GlobalAlloc` 的定义可以在[这里](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html)找到，其接口如下：

```rust
pub unsafe trait GlobalAlloc {
    // Required methods
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    // Provided methods
    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 { ... }
    unsafe fn realloc(
        &self,
        ptr: *mut u8,
        layout: Layout,
        new_size: usize
    ) -> *mut u8 { ... }
}
```

`GlobalAlloc` 定义了 `alloc` 和 `dealloc` 两个方法，分别用于分配和释放内存。你写的 heap allocator 需要实现 `GlobalAlloc` 这个 trait，且具体有以下要求：

1. `alloc`：返回一个指向分配的内存的一个字节的指针，如果分配失败则返回 `null`。
2. `dealloc`：释放之前分配的内存。**要求这块内存之前是通过 `alloc` 分配的**。

一个不能用的示例如下：

```rust
pub struct Dummy;

unsafe impl GlobalAlloc for Dummy {
    unsafe fn alloc(&self, _layout: Layout) -> *mut u8 {
        null_mut()
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        panic!("dealloc should be never called")
    }
}
```

以上代码参考自[这篇文章](https://os.phil-opp.com/heap-allocation/#the-globalalloc-trait)，更多的信息也可参考之。

## Rust 中的堆数据结构

在 kernel 中能使用的数据结构来自于 `alloc` 这个 crate，其提供的数据结构与 `std` 中的数据结构类似但并不完全相同，且实现方式也可能不一致。里面可能有用的 modules 列举如下：

- [box](https://doc.rust-lang.org/alloc/boxed/index.html)：提供了 `Box` 这个数据结构，用于分配在堆上的数据。
- [collections](https://doc.rust-lang.org/alloc/collections/index.html)：提供了 `BinaryHeap`、`BTreeMap`、`BTreeSet`、`LinkedList`、`VecDeque`。
- [rc](https://doc.rust-lang.org/alloc/rc/index.html)：提供了 `Rc` 和 `Weak`。
- [str](https://doc.rust-lang.org/alloc/str/index.html)：提供了 `str`。
- [string](https://doc.rust-lang.org/alloc/string/index.html)：提供了 `String`。
- [vec](https://doc.rust-lang.org/alloc/vec/index.html)：提供了 `Vec`。

## Buddy Allocator

### 算法介绍

> The buddy system assumes that memory is of size  2N  for some integer  N . Both free and reserved blocks will always be of size  2k  for  k <= N . At any given time, there might be both free and reserved blocks of various sizes. The buddy system keeps a separate list for free blocks of each size. There can be at most  2k  such lists, because there can only be  2k  distinct block sizes.
>
> When a request comes in for m words, we first determine the smallest value of k such that  2k >= m  . A block of size  2k  is selected from the free list for that block size if one exists. The buddy system does not worry about internal fragmentation: The entire block of size  2k  is allocated.
>
> If no block of size  2k  exists, the next larger block is located. This block is split in half (repeatedly if necessary) until the desired block of size  2k  is created. Any other blocks generated as a by-product of this splitting process are placed on the appropriate freelists.
>
> The disadvantage of the buddy system is that it allows internal fragmentation. For example, a request for 257 words will require a block of size 512. The primary advantages of the buddy system are:
>
>  1. there is less external fragmentation;
>  2. search for a block of the right size is cheaper than, say, best fit because we need only find the first available block on the block list for blocks of size  2k ; and
>  3. merging adjacent free blocks is easy.
>
> The reason why this method is called the buddy system is because of the way that merging takes place. The buddy for any block of size  2k  is another block of the same size, and with the same address (i.e., the byte position in memory, read as a binary value) except that the kth bit is reversed. For example, the block of size 8 with beginning address 0000 in the figure below, has buddy with address 1000. Likewise the block of size 4 with address 0000 has buddy 0100. If free blocks are sorted by address value, the buddy can be found by searching the correct block size list. Merging simply requires that the address for the combined buddies be moved to the freelist for the next larger block size.

以上片段参考自 [Memory Management Tutorial Section 2.3 - Buddy Methods](https://research.cs.vt.edu/AVresearch/MMtutorial/buddy.php)。

还有一些别的参考材料：[wikipedia](https://en.wikipedia.org/wiki/Buddy_memory_allocation)，[recore wiki](https://github.com/Celve/recore/wiki/Allocator#buddy-allocator)。

### 如何开始

首先，我们需要划分一块区域作为内存池。一种简单的方式就是直接使用 `static` 变量，即全局变量：

```rust
static mut KERNEL_HEAP_SPACE: [u8; KERNEL_HEAP_SIZE] = [0; KERNEL_HEAP_SIZE];
```

然后在这段空间上实现 buddy allocator。
