# Service

## Microkernel

Service 是 microkernel 的核心组件之一。我们先来看一下 microkernel 的定义（摘自维基百科）：

> Microkernel 的设计理念，是将系统服务的实现，与系统的基本操作规则区分开来。它实现的方式，是将核心功能模块化，划分成几个独立的进程，各自运行，这些进程被称为服务（service）。所有的 service，都运行在不同的地址空间。只有需要绝对特权的进程，才能在 S mode 下运行，其余的进程则在 U mode 运行。
>
> 这样的设计，使内核中最核心的功能，设计上变的更简单。需要特权的部分，只有基本的线程管理，内存管理和进程间通信等，这个部分，由一个简单的硬件抽象层与关键的系统调用组成。其余的服务进程，则移至用户空间。
>
> 让服务各自独立，可以减少系统之间的耦合度，易于实现与调试，也可增进可移植性。它可以避免单一组件失效，而造成整个系统崩溃，内核只需要重启这个组件，不致于影响其他服务器的功能，使系统稳定度增加。同时，操作系统也可以视需要，抽换或新增某些服务进程，使功能更有弹性。
>
> 因为所有服务进程都各自在不同地址空间运行，因此在微核心架构下，不能像宏内核一样直接进行函数调用。在微核心架构下，要建立一个进程间通信机制，通过消息传递的机制来让服务进程间相互交换消息，调用彼此的服务，以及完成同步。采用主从式架构，使得它在分布式系统中有特别的优势，因为远程系统与本地进程间，可以采用同一套进程间通信机制。

## Overview

在我们的系统中，为了简单起见，我们一共需要实现以下几个 service：

1. Scheduler (S mode)
2. Drivers (U mode) *Undetermined*
3. File system (U mode)
4. Process manager (U mode)

我们可以用一张图来总结上述 services 之间的关系：

![overview](assets/overview.png)

这张图描述了一个用户程序进行 `sys_open` 的全过程。

在左上角的 **app** 处：

- 用户程序通过 `syscall` 进入内核态，并传递参数，譬如 `syscall number`，`path`，`flags` 等；
- 内核态的 syscall handler 会根据 `syscall number` 调用对应的服务；在这个例子中，我们需要获得 `path` 对应的文件，所以，内核线程通过 IPC 调用 **file system serivce**；
- 由于我们需要等待 file system service 的响应，所以内核线程会进入等待状态，并切换到 **scheduler**。

到了 **scheduler**：

- **Scheduler** 会根据当前的任务队列，依照内部算法，选择下一个任务；
- 由于我们对于文件系统的调用是异步的，所以 **scheduler** 选择的下一个任务不一定是 **file system service**，这里假设我们在第 A 个任务选择了 **file system service**；同理，我们假设在第 B 个任务选择了 **driver service**，以此类推。

到了 **file system**：

- 我们从内核线程切换到用户线程（**file system service** 是 U mode 的）, **file system service** 会尝试根据 **path** 找到对应的文件，并返回给内核线程；
- 但在这个过程中，**file system service** 需要访问 block device 来读取一些数据以找到对应的文件，于是用户线程通过 IPC 调用 **driver service**；
- 其后用户线程切换到内核线程，内核线程随后切换到 **scheduler**。

到了 **scheduler**：

- **Scheduler** 会根据当前的任务队列，依照内部算法，选择下一个任务；限于篇幅，以后的 **scheduler** 部分不再赘述。

到了 **driver**：

- 首先切换到应用线程，`block device` 会读取对应的 `block`，并返回给 **file system service**；
- 其后用户线程切换到内核线程，内核线程随后切换到 **scheduler**。

到了 **file system**：

- 在与 **driver service** 交互找到对应的文件后，返回给 **app** 对应的内核线程。

到了 **app**：

- 内核线程在找到对应的文件后，通过 IPC 联系 **process manager** 准备创建一个新的 file descriptor。

到了 **process manager**：

- **process manager** 会为 **app** 创建一个新的 file descriptor，并返回给 **app**。

到了 **app**：

- **app** 中的内核线程会收到 **process manager** 的返回，并返回给用户程序。
