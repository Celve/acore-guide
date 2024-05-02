# 进程

进程的结构如下所示：

```rust
struct Process {
    pid: usize, // process ID
    vmspace: VmSpace, // virtual memory space
    pagetable: VirAddr, // page table
    state: ProcessState, // process state
    parent: Option<Weak<Process>>, // parent process
    children: Vec<Arc<Process>>, // children processes
    fdtable: FdTable, // file descriptor table
}
```

在微内核的设计中，process 在 process manager 中进行维护。process manager 是一个服务，它负责创建、销毁和管理 processes。Process manager 也负责为 process 分配资源，比如文件描述符等。

需要注意的是，page table 的 allocation 必须在内核态中进行（因为需要操作页表，其中包含了物理地址）。

## 与任务的关系

在 [切换内核进程](switch.md) 中，我们提到了 [任务](task.md) 。需要明确的是，进程与任务虽有交集，但本质上是两个不同的概念。

**进程**作为资源分配的单元，相对独立，并且在操作系统中具有各自的内存和系统资源。它们是系统资源管理和分配的基础，使得多个程序能在多任务操作系统中同时运行而互不干扰。

而**任务**通常指被调度器调度的执行单元。**任务**包含了一切与执行调度相关的信息。
