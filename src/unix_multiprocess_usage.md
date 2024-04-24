# Unix 多进程用例

在这一章节中，我们会介绍 Unix 的多进程使用方法。Unix 的系统调用形式非常精简，因此经常被用于教学。我们会通过一个简单的例子来展示如何使用 Unix 的系统调用来创建进程、等待进程结束、以及进程间通信。以下代码均作示例用，与我们最终的实现无关。

## 系统调用

Unix 多线程相关的系统调用主要有：

- `fork`：创建一个新的进程。
- `exec`：加载一个新的程序。
- `exit`：结束当前进程。
- `wait`：等待一个子进程结束。

### fork

`fork` 系统调用比较有趣，在内核看来，它表示应用需要创建一个新的子进程，其地址空间与父进程相同，下一个执行的指令也相同，唯一不同的是返回值。父进程中的返回值是子进程的 PID，而子进程中的返回值是 0。这样，父进程和子进程可以通过返回值来区分自己是父进程还是子进程。如果 `fork` 失败，将会向调用进程返回 -1。

在用户看来，`fork` 会在原来程序的基础上，复制出一个新的子进程，这个子进程会从 `fork` 之后的指令开始执行。`fork` 的返回值用来告诉用户当前是父进程还是子进程。

在程序看来，`fork` 调用后，会返回一个 `int` 用于区分当前是父进程还是子进程。而且，从程序的视角看，我们无需太过考虑多进程这件事，因为程序的自己永远都在自己的地址空间上运行。

你可能对上面的表述有一些疑惑，下面我们将用一个例子来展示 `fork` 的用法。

```c
#include <stdio.h>

int main() {
    int pid = fork();
    if (pid < 0) {
        printf("fork failed\n");
    } else if (pid == 0) {
        printf("I am child process\n");
    } else {
        printf("I am parent process\n");
    }
    return 0;
}
```

在这个例子中，如果 `fork` 成功，父进程会输出 `I am parent process`，而子进程会输出 `I am child process`。

### exec

`fork` 虽然可以创建出一个新的进程，但是这个进程的地址空间和原来是一样的，因此如果我们要运行一个完全不一样的程序，那么 `fork` 的功能就不够了。这时候我们就需要 `exec` 系统调用。

`exec` 会加载一个新的程序到当前进程的地址空间中，并且开始执行这个程序，也就是「变身」。通常 `exec` 不会返回，除非出现了错误。

`exec` 是一类系统调用，包括 `execl`、`execv`、`execle`、`execve`、`execlp`、`execvp` 等（具体请自己查阅 manual page，如执行 `man 2 execve`）。这些系统调用的区别在于参数的传递方式和搜索路径的不同。

下面是一个使用 `execve` 的例子：

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    char *args[] = {"/bin/ls", "-l", NULL};
    execve("/bin/ls", args, NULL);
    return 0;
}
```

在这个例子中，我们使用 `execve` 来执行 `/bin/ls -l`，这个程序会列出当前目录下的文件。

### exit

`exit` 系统调用用于结束当前进程。`exit` 有一个输入参数用于表达退出的状态。

### wait

`wait` 系统调用用于等待一个子进程结束。如果当前没有子进程，`wait` 会立即返回。如果有子进程结束，`wait` 会返回这个子进程的 PID。

如果在子进程退出时父进程没有调用 `wait`，那么子进程会变成僵尸进程，会有一些核心资源无法回收。僵尸进程会占用系统资源，因此我们应该在子进程退出时调用 `wait` 来回收资源。另一方面，如果父进程先退出了，那么子进程会成为孤儿进程，这时候子进程会被 `init` 进程接管，`init` 进程会调用 `wait` 来回收资源。`init` 进程一旦退出，就会造成 kernel panic。

下面是一个使用 `wait` 的例子：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    int pid = fork();
    if (pid < 0) {
        printf("fork failed\n");
    } else if (pid == 0) {
        printf("I am child process\n");
        sleep(1);
    } else {
        wait(NULL);
        printf("I am parent process\n");
    }
    return 0;
}
```

在上面的例子中，父进程会等待子进程结束后再输出 `I am parent process`。

## Unix 产生子进程的方式

关于多进程，我们的需求通常是产生一个子进程，而这个子进程通常是和父进程不同的。

利用我们上面介绍的四个系统调用，Unix 中我们通常会使用下面的方法来满足上述需求：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    int pid = fork();
    if (pid < 0) {
        printf("fork failed\n");
    } else if (pid == 0) {
        // child process
        char *args[] = {"/bin/ls", "-l", NULL};
        execve("/bin/ls", args, NULL);
    } else {
        // 父进程
        wait(NULL);
    }
    exit(0);
}
```

如你所见，我们通常会先使用 `fork` 产生一个子进程，然后在子进程中使用 `exec` 来加载一个新的程序。父进程通常会使用 `wait` 来等待子进程结束。

在这个例子中，父进程会产生一个子进程，然后等待子进程结束。子进程会执行 `/bin/ls -l`。
