# Supervisor Binary Interface (SBI)

## SBI 原理及简介

### 引子

我们都知道如何写一个 hello world 程序。但是，为什么这个程序可以在终端上打印 hello world 呢？

解答这个问题前，我们需要先把问题简化一些。让我们回到上个世纪，那个真的用打印机作为输出的年代。

打印机的具体实现这里我们不关心，我们只需要知道，操作系统往一个特定的地址写一个特定的数据，打印机终端就能执行一个操作，如打印字符 h 或者打印机换到下一行，这叫做 MMIO (Memory mapping I/O)。（如果你实在是特别好奇为什么这样是可以的，下面列举了一种可行的实现。）

<details>
<summary><strong>点击查看一种可行的 MMIO 实现</strong></summary>
当 CPU 执行到读内存地址时，（在 MMU 翻译到物理地址后，）会向主板上的芯片发送写请求，主板上的芯片会区分地址对应的物理设备（这个设备可能是内存，也可能是输入输出设备，如打印机）。当对应的物理设备是打印机时，会采取硬件所要求的方式正确处理请求。
</details>

所以说，操作输入输出设备，本质上就是读/写 MMIO 地址（这个地址和一般的地址的形式是一样的）。

### 什么是 SBI？

从字面意思上，SBI 就是 supervisor 的二进制接口。因此，SBI 的任务就是要为 supervisor（也就是内核）提供支持。

上面的打印字符，就是 SBI 需要实现的功能，因为内核本身并没有这个能力，我们需要通过操作外部设备实现。

除此以外，内核还需要进入 machine mode 做一些操作，那么这个也需要 SBI 提供支持。

### SBI 的需求

了解了 SBI 的作用，我们现在需要大概分析一下 SBI 需要提供哪些功能。

SBI 作为 supervisor 的接口，其本身主要是在为 supervisor（也就是内核）服务。

内核大致需要这些接口：（具体要实现的内容可以不一样，取决于你的内核需要什么）

- IO（在控制台输出字符、从控制台读取字符）；
- 其他硬件操作（如关机）；
- 只能在 machine mode 操作的设置（如计时器、修改一些特权寄存器的特定比特）。

## SBI 实现细节

### UART

不同于 rCore-Tutorial，在 ACore 中，我们要求**自己**实现 SBI 的功能。第一阶段中，我们需要实现一个十分简易的 SBI 来在命令行中打印字符。在此之前，我们首先需要了解 UART。它是一种串行通信设备，用于在计算机和外部设备之间传输数据。在[维基百科](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter)中有着更加详细的介绍。在 ACore 中，我们使用的是 QEMU 的 virt 平台，它的串口设备是一个 NS16550A UART。我们需要实现的 SBI 功能是通过 UART 将字符打印到命令行。

我们利用 MMIO 来和串口设备进行通信。[引子](#引子)对于 MMIO 已经有了简单的介绍，我们在这里再补充一些细节。在正式和串口设备通信之前，我们要初始化串口设备，这个过程需要写入一些特定的值到串口设备的寄存器中，而这些寄存器就是通过 MMIO 的方式映射在内存中的。通过这种方式，我们可以与串口设备协定一些最基础的信息，譬如我们是否开启 FIFO 队列（通过 FCR 寄存器），我们是否开启中断（通过 IER 寄存器）等等。

在初始化完成后，我们就可以通过读写串口设备的寄存器来进行输入输出了。我们可以通过读取 RBR 寄存器来获取串口设备接收到的字符，通过写入 THR 寄存器来发送字符，通过读取 LSR 寄存器来获取串口设备的状态。而这些都依赖于 MMIO。

在 QEMU 模拟的 virt 计算机中串口设备寄存器的 MMIO 起始地址为 `0x10000000`，在下表被称为 `base`。连续的 8 个 bit 组成一个寄存器。下表给出了 UART 中每个寄存器的地址和基本含义。
| I/O Port | Read (DLAB=0)  | Write (DLAB=0) | Read (DLAB=1) | Write (DLAB=1) |
|----------|----------------|----------------|---------------|----------------|
| base     | **RBR** receiver buffer | **THR** transmitter holding | **DLL** divisor latch LSB | **DLL** divisor latch LSB |
| base+1   | **IER** interrupt enable | **IER** interrupt enable | **DLM** divisor latch MSB | **DLM** divisor latch MSB |
| base+2   | **IIR** interrupt identification | **FCR** FIFO control | **IIR** interrupt identification | **FCR** FIFO control  |
| base+3   | **LCR** line control | **LCR** line control | **LCR** line control | **LCR** line control |
| base+4   | **MCR** modem control | **MCR** modem control | **MCR** modem control | **MCR** modem control |
| base+5   | **LSR** line status | *factory test* | **LSR** line status | *factory test* |
| base+6   | **MSR** modem status | *not used* | **MSR** modem status | *not used* |
| base+7   | **SCR** scratch | **SCR** scratch | **SCR** scratch | **SCR** scratch |

介于篇幅有限，寄存器的详细含义以及如何设置请参考[这篇博客](https://www.lammertbies.nl/comm/info/serial-uart)。

#### UART 初始化

UART 的初始化可以参考 [xv6](https://github.com/mit-pdos/xv6-riscv/blob/f5b93ef12f7159f74f80f94729ee4faabe42c360/kernel/uart.c#L53)，[recore](https://github.com/Celve/recore/blob/dd95657ba2f0450df904d88488bf0d2c171d09ed/kernel/src/drivers/uart.rs#L130) 和 [uart_16550](https://github.com/rust-osdev/uart_16550/blob/378d468b5f80effc0b53f537fabc2fd73d16449e/src/mmio.rs#L40)。

#### 使用 UART 进行输入输出

使用 UART 输入输出主要和 RBR，THR 和 LSR 寄存器进行交互，具体依旧可以参考[这篇博客](https://www.lammertbies.nl/comm/info/serial-uart)。

之后建议参照 [rCore-Tutorial](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter1/6print-and-shutdown-based-on-sbi.html#id2) 中的介绍，创建 `print!` 宏与 `println!` 宏，以方便后续的调试。

#### 使用 QEMU 进行调试

因为我们不使用 rCore-Tutorial 中的 SBI，所以 QEMU 的启动参数需要做一些调整：

```bash
qemu-system-riscv64 \
  -machine virt \
  -nographic \
  -bios none \
  -device loader,file=$KERNEL_BIN,addr=$KERNEL_ENTRY_PA \
  -s -S
```

`KERNEL_BIN` 表示编译生成的内核二进制文件，`KERNEL_ENTRY_PA`表示内核的入口地址。`-s -S` 参数表示启动 QEMU 时暂停 CPU，等待 GDB 连接。默认开启 1234 端口。可以利用 GDB 连接 QEMU，进行调试：

```bash
riscv64-unknown-elf-gdb \
  -ex "file $KERNEL_ELF" \
  -ex "set arch riscv:rv64" \
  -ex "target remote localhost:1234"
```

`KERNEL_ELF` 表示编译生成的内核 ELF 文件。
