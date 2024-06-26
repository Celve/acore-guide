# Schedule

以下是推荐的作业安排，每周的任务是一个大致的估计，具体的时间可能会有所调整。

|  Weeks  | Tasks  | Reading  |
| ------------ | ------------- | -------------- |
| 第二周  | 选大作业/小作业 | rCore 第零章 |
| 第三周-第四周  | 基于 SBI 写 hello world | rCore 第一章 |
| 第五周-第七周  | 页表和特权级转化 (m-mode -> s-mode)，实现 buddy allocator | rCore 第二章、第四章 |
| 第八周-第九周  | 创建单进程（s-mode -> u-mode）和 trap | rCore第五章 |
| 第十周-第十二周  | process manager，scheduler | rCore 第三章 |
| 第十三周-第十六周  | drivers, FS，shell | rCore 第六章、第九章 |
| 第十七周  | IPC | rCore 第七章 |

**注意：本文档不能代替 RISCV64 特权指令集文档，如对 RISCV 特权指令集相关内容有不清楚的地方，请参阅 [RISCV 特权指令集文档][riscv-privileged-spec]。**

[riscv-privileged-spec]: https://github.com/riscv/riscv-isa-manual/releases/tag/Priv-v1.12
