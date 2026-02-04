# 引言

## 本章导读

在第六章中，我们实现了文件系统，为进程提供了文件描述符机制，支持进程读写文件、使用管道进行进程间通信等功能。本章将在第六章的基础上，为操作系统添加**信号（Signal）**机制的支持。

信号是 Unix/Linux 系统中一种重要的进程间通信机制。它允许一个进程向另一个进程发送异步通知，告知对方某个事件已经发生。例如，当用户按下 Ctrl+C 时，终端会向当前前台进程发送 SIGINT 信号，通知进程中断当前操作；当进程访问非法内存地址时，内核会向进程发送 SIGSEGV 信号，导致进程终止。

本章的目标是实现一个基本的信号机制，包括：

- 信号的发送和接收机制
- 信号处理函数的注册和调用
- 信号掩码的屏蔽功能
- 特殊信号的处理（如 SIGKILL、SIGSTOP）
- 信号在 fork 和 exec 时的行为

## 实践体验

在 `rCore-Tutorial-in-single-workspace` 项目中运行本章代码（ `DEBUG` 类型的调试输出过多，建议使用 `LOG=INFO` 过滤输出）：

```bash
$ LOG=INFO cargo qemu --ch 7
```

运行后，进入 shell 程序，可以运行信号机制的简单测例 `sig_simple`：

```
>> sig_simple
pid = 2
signal_simple: sigaction
signal_simple: kill
user_sig_test succsess
signal_simple: Done
Shell: Process 2 exited with code 0
>>
```

这个测例展示了信号的基本使用流程：首先通过 `sigaction` 系统调用注册一个信号处理函数，然后通过 `kill` 系统调用向当前进程发送信号，信号处理函数被调用并执行。

另一个测例 `sig_ctrlc` 展示了如何处理 SIGINT 信号（通常由 Ctrl+C 触发）：

```
>> sig_ctrlc
sig_ctrlc starting....  Press 'ctrl-c' or 'ENTER'  will quit.
sig_ctrlc: sigaction
sig_ctrlc: getchar....
Got Char  3
signal_handler: caught signal SIGINT, and exit(1)
Shell: Process 2 exited with code 1
>>
```

当用户按下 Ctrl+C（字符码为 3）时，内核会向进程发送 SIGINT 信号，进程注册的信号处理函数被调用，最终进程退出。

## 本章代码树

```
├── ch7（内核实现）
│   ├── Cargo.toml（配置文件，新增 signal 和 signal-impl 依赖）
│   └── src（内核源代码）
│       ├── main.rs（内核主函数，包括信号相关系统调用实现）
│       ├── process.rs（进程结构，集成信号模块）
│       ├── processor.rs（进程管理器）
│       ├── fs.rs（文件系统相关）
│       └── virtio_block.rs（块设备驱动）
├── signal-defs（信号定义模块）
│   └── src
│       └── lib.rs（信号编号 SignalNo 和信号处理函数 SignalAction 的定义）
├── signal（信号接口模块）
│   └── src
│       ├── lib.rs（Signal trait 定义）
│       └── signal_result.rs（信号处理结果 SignalResult）
├── signal-impl（信号实现模块）
│   └── src
│       ├── lib.rs（SignalImpl 实现）
│       ├── signal_set.rs（信号集合 SignalSet）
│       └── default_action.rs（默认行为 DefaultAction）
├── syscall（系统调用模块）
│   └── src
│       └── ...（包含信号相关系统调用的定义）
├── user（用户程序）
│   └── src/bin（测试用例）
│       ├── sig_simple.rs（简单信号测试）
│       ├── sig_ctrlc.rs（SIGINT 信号处理测试）
│       └── ...
├── ...
```
