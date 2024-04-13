# 系统调用接口

在编写用户程序时，最简单的调试方式就是输出信息。但是，用户程序如何才能输出信息呢?答案是通过系统调用。

系统调用是操作系统内核提供给用户程序的一组标准接口。通过系统调用，用户程序可以请求内核提供各种服务，例如文件读写、进程管理、网络通信等。系统调用是用户程序与内核之间通信的桥梁。

在[系统调用](syscall.md)这一节中，我们已经讨论了系统调用的实现原理。在这一节中，我们将讨论用户程序如何调用系统调用。

## 实现接口

以 `SYSCALL_WRITE` 为例，我们来看看如何在用户态实现系统调用接口。当程序运行在用户态(U mode)时，可以通过 `ecall` 指令触发系统调用。因此，用户态的系统调用接口本质上就是通过 `ecall` 指令进入内核态，由内核处理系统调用请求。
`ecall` 指令会触发一个特殊的异常(exception)，称为环境调用(environment call)。内核的异常处理程序可以识别这种异常，进一步根据 `a7` 寄存器的值确定具体的系统调用类型，并调用相应的处理函数。而在用户态，我们只需要将系统调用号放在 `a7` 寄存器，其他参数依次放在 `a0~a6` 寄存器中，然后执行 `ecall` 指令即可。下面是一个通用的系统调用接口实现:

```rust
fn syscall(id: usize, args: [usize; 6]) -> isize {
    let mut ret: isize;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x13") args[3],
            in("x14") args[4],
            in("x15") args[5],
            in("x17") id,
        );
    }
    ret
}
```

其中，`id` 表示系统调用号，需要与内核定义一致。`args` 是一个最多包含 6 个参数的数组，传递给内核的系统调用处理函数。

有了这个通用接口，我们可以进一步封装出更易用的系统调用函数。以 `SYSCALL_WRITE` 为例:

```rust
pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len(), 0, 0, 0, 0])  
}
```

类似的，`SYSCALL_READ` 系统调用的封装如下：

```rust
pub fn sys_read(fd: usize, buffer: &mut [u8]) -> isize {
    syscall(
        SYSCALL_READ,
        [fd, buffer.as_mut_ptr() as usize, buffer.len()],
    )
}
```

## 用户库

基于 `sys_write` 和 `sys_read` 等系统调用封装函数，我们可以进一步实现一些常用的用户库函数，使得用户程序更方便地使用系统服务。例如，实现 `putchar` 和 `getchar` 函数：

```rust
const STDIN: usize = 0;
const STDOUT: usize = 1;

pub extern "C" fn putchar(c: u8) {
    sys_write(STDOUT, &[c]);
}

pub extern "C" fn getchar() -> u8 {
    let mut buf = [0u8; 1];
    sys_read(STDIN, &mut buf);
    buf[0]
}
```

这样，用户程序就可以直接调用 `putchar` 和 `getchar` 来输出/输入单个字符，而无需关心底层的系统调用细节。
