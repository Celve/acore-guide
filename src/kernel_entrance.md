# 内核入口

我们知道，当内核被引导时，我们唯一可以确定的通用寄存器是 pc，包括 sp 在内的其他可写通用寄存器是没有意义的。也就是说，我们甚至没有函数栈。因此，在执行到内核的第一条指令时，我们无法使用任何常规的高级语言，必须使用机器码或者汇编。并利用这些代码设置函数栈。

More coming soon...
