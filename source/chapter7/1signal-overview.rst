信号机制概述
============================================

信号的概念和作用
--------------------------------------------

信号（Signal）是 Unix/Linux 系统中一种重要的进程间通信机制。它允许一个进程向另一个进程发送异步通知，告知对方某个事件已经发生。信号类似于硬件中断，但发生在用户态，是软件层面的中断机制。

信号的主要用途包括：

- **进程控制**：父进程可以通过信号控制子进程的行为，如终止子进程（SIGTERM）、暂停子进程（SIGSTOP）、恢复子进程（SIGCONT）等
- **异常处理**：当进程发生异常时，内核会向进程发送相应的信号，如访问非法内存地址时发送 SIGSEGV，执行非法指令时发送 SIGILL
- **用户交互**：用户可以通过键盘输入（如 Ctrl+C）向进程发送信号，实现进程的中断和终止
- **进程间通信**：进程可以通过信号通知其他进程某个事件已经发生

信号的生命周期
--------------------------------------------

信号从产生到处理完成，经历了以下几个阶段：

1. **信号产生**：信号可以由内核、其他进程或用户操作产生。例如，当进程访问非法内存时，内核会产生 SIGSEGV 信号；当进程调用 ``kill`` 系统调用时，会产生指定的信号。

2. **信号发送**：信号被发送到目标进程。每个进程都有一个信号队列，用于存储已收到但尚未处理的信号。

3. **信号接收**：目标进程接收到信号，将信号添加到信号队列中。

4. **信号处理**：当进程从内核态返回用户态时（如系统调用返回），内核会检查进程是否有待处理的信号。如果有，则根据信号的类型和处理函数设置，决定如何处理信号：
   - 如果进程注册了信号处理函数，则调用该函数
   - 如果没有注册处理函数，则执行默认行为（通常是终止进程或忽略信号）

5. **信号处理完成**：如果调用了用户注册的处理函数，处理函数执行完毕后，通过 ``sigreturn`` 系统调用恢复原来的上下文，继续执行被中断的代码。

信号模块的架构设计
--------------------------------------------

为了保持代码的模块化和可扩展性，我们将信号机制的实现分为三个独立的 crate：

.. code-block::

   signal-defs
      ↑
      │ (依赖)
      ├── signal
      └── syscall
      ↑
      │ (实现)
      signal-impl

- **signal-defs**：定义用户程序和内核都需要使用的信号相关数据结构，包括信号编号 ``SignalNo`` 和信号处理函数 ``SignalAction``。这个模块被 ``signal`` 和 ``syscall`` 模块共同依赖，避免了循环依赖的问题。

- **signal**：定义信号机制的接口 trait ``Signal``，它提供了信号管理的核心方法。这个设计使得信号机制的实现可以独立于内核的其他部分，提高了代码的模块化程度。

- **signal-impl**：提供 ``Signal`` trait 的一个具体实现 ``SignalImpl``，它管理每个进程的信号状态，包括已收到的信号、信号掩码、信号处理函数等。

这种设计的好处是：

- **解耦合**：信号机制的接口和实现分离，内核的其他部分只需要依赖 ``signal`` 模块的接口，而不需要关心具体的实现细节
- **可扩展性**：如果需要修改信号机制的实现，只需要修改 ``signal-impl`` 模块，而不需要修改其他模块
- **可测试性**：可以轻松地为 ``Signal`` trait 编写 mock 实现，方便进行单元测试

signal-defs 模块
--------------------------------------------

``signal-defs`` 模块定义了信号机制的基础数据结构，位于 ``rCore-Tutorial-in-single-workspace/signal-defs/src/lib.rs``。

信号编号 SignalNo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``SignalNo`` 是一个枚举类型，定义了所有支持的信号编号：

.. code-block:: rust
   :linenos:
   :caption: signal-defs/src/lib.rs

   #[repr(u8)]
   pub enum SignalNo {
       ERR = 0,
       SIGHUP = 1,    // 挂起信号
       SIGINT = 2,    // 中断信号（通常由 Ctrl+C 触发）
       SIGQUIT = 3,   // 退出信号
       SIGILL = 4,    // 非法指令
       SIGTRAP = 5,   // 跟踪陷阱
       SIGABRT = 6,   // 异常终止
       SIGBUS = 7,    // 总线错误
       SIGFPE = 8,    // 浮点异常
       SIGKILL = 9,   // 强制终止（不能被捕获或忽略）
       SIGUSR1 = 10,  // 用户自定义信号 1
       SIGSEGV = 11,  // 段错误（访问非法内存）
       SIGUSR2 = 12,  // 用户自定义信号 2
       SIGPIPE = 13,  // 管道破裂
       SIGALRM = 14,  // 闹钟信号
       SIGTERM = 15,  // 终止信号
       SIGSTOP = 19,  // 暂停进程（不能被捕获或忽略）
       SIGCONT = 18,  // 继续执行被暂停的进程
       // ... 其他信号
   }

每个信号都有其特定的含义和用途。例如：

- **SIGINT**：通常由用户按下 Ctrl+C 触发，用于中断当前进程
- **SIGKILL**：强制终止进程，不能被捕获或忽略，是终止进程的最可靠方式
- **SIGSTOP**：暂停进程的执行，不能被捕获或忽略
- **SIGCONT**：恢复被暂停的进程
- **SIGUSR1/SIGUSR2**：用户自定义信号，可以由应用程序自由使用

信号处理函数 SignalAction
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``SignalAction`` 结构体定义了信号处理函数的信息：

.. code-block:: rust
   :linenos:
   :caption: signal-defs/src/lib.rs

   #[repr(C)]
   #[derive(Debug, Clone, Copy, Default)]
   pub struct SignalAction {
       pub handler: usize,  // 信号处理函数的地址
       pub mask: usize,    // 在处理该信号时，需要屏蔽的信号掩码
   }

- **handler**：信号处理函数的地址。当进程收到信号时，如果注册了处理函数，内核会将程序计数器（PC）设置为该地址，从而调用处理函数。

- **mask**：在处理该信号时，需要屏蔽的信号掩码。这是一个位掩码，每一位对应一个信号。当处理函数执行时，被屏蔽的信号不会被处理，直到处理函数返回。

信号掩码的作用是防止信号处理函数被其他信号中断，保证信号处理的原子性。例如，如果进程正在处理 SIGUSR1 信号，此时又收到了另一个 SIGUSR1 信号，如果该信号没有被屏蔽，可能会导致处理函数被递归调用，造成不可预期的行为。

signal 模块
--------------------------------------------

``signal`` 模块定义了信号机制的接口 trait ``Signal``，位于 ``rCore-Tutorial-in-single-workspace/signal/src/lib.rs``。

Signal trait 的核心方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust
   :linenos:
   :caption: signal/src/lib.rs

   pub trait Signal: Send + Sync {
       /// fork 时继承信号处理函数和掩码
       fn from_fork(&mut self) -> Box<dyn Signal>;
       
       /// exec 时清除信号处理函数（但保留掩码）
       fn clear(&mut self);
       
       /// 添加一个信号到信号队列
       fn add_signal(&mut self, signal: SignalNo);
       
       /// 是否当前正在处理信号
       fn is_handling_signal(&self) -> bool;
       
       /// 设置信号处理函数
       fn set_action(&mut self, signum: SignalNo, action: &SignalAction) -> bool;
       
       /// 获取信号处理函数
       fn get_action_ref(&self, signum: SignalNo) -> Option<SignalAction>;
       
       /// 更新信号掩码，返回旧的掩码
       fn update_mask(&mut self, mask: usize) -> usize;
       
       /// 处理信号，返回处理结果
       fn handle_signals(&mut self, current_context: &mut LocalContext) -> SignalResult;
       
       /// 从信号处理函数返回，恢复原来的上下文
       fn sig_return(&mut self, current_context: &mut LocalContext) -> bool;
   }

这些方法涵盖了信号管理的各个方面：

- **from_fork()**：当进程调用 ``fork`` 创建子进程时，子进程需要继承父进程的信号处理函数和掩码。这是因为子进程是父进程的副本，应该具有相同的信号处理行为。

- **clear()**：当进程调用 ``exec`` 执行新程序时，需要清除所有信号处理函数（但保留信号掩码）。这是因为新程序可能有自己的信号处理逻辑，不应该继承旧程序的处理函数。

- **add_signal()**：将信号添加到进程的信号队列中。当其他进程调用 ``kill`` 系统调用时，内核会调用此方法。

- **handle_signals()**：处理信号的核心方法。当进程从内核态返回用户态时，内核会调用此方法检查并处理待处理的信号。该方法会修改用户程序的上下文（如程序计数器），使其跳转到信号处理函数。

- **sig_return()**：当信号处理函数执行完毕后，通过 ``sigreturn`` 系统调用恢复原来的上下文，继续执行被中断的代码。

SignalResult 枚举
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``SignalResult`` 枚举定义了信号处理的结果，位于 ``rCore-Tutorial-in-single-workspace/signal/src/signal_result.rs``：

.. code-block:: rust
   :linenos:
   :caption: signal/src/signal_result.rs

   pub enum SignalResult {
       NoSignal,              // 没有信号需要处理
       IsHandlingSignal,      // 正在处理信号，无法接受其他信号
       Ignored,               // 信号被忽略
       Handled,               // 信号已处理，需要修改用户上下文
       ProcessKilled(i32),    // 进程需要被终止，返回退出码
       ProcessSuspended,      // 进程需要被暂停
   }

内核根据 ``handle_signals()`` 返回的结果，决定下一步的操作：

- **NoSignal**：没有待处理的信号，正常返回用户态
- **IsHandlingSignal**：正在处理信号，忽略新收到的信号
- **Handled**：信号已处理，用户上下文已被修改，直接返回用户态执行处理函数
- **ProcessKilled**：进程需要被终止，内核会调用进程退出逻辑
- **ProcessSuspended**：进程需要被暂停，内核会将其从调度队列中移除

signal-impl 模块
--------------------------------------------

``signal-impl`` 模块提供了 ``Signal`` trait 的一个具体实现 ``SignalImpl``，位于 ``rCore-Tutorial-in-single-workspace/signal-impl/src/lib.rs``。

SignalImpl 数据结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   pub struct SignalImpl {
       /// 已收到的信号集合
       pub received: SignalSet,
       /// 信号掩码（被屏蔽的信号不会被处理）
       pub mask: SignalSet,
       /// 正在处理的信号状态
       pub handling: Option<HandlingSignal>,
       /// 信号处理函数数组
       pub actions: [Option<SignalAction>; MAX_SIG + 1],
   }

- **received**：使用 ``SignalSet`` 类型存储已收到但尚未处理的信号。``SignalSet`` 是一个位集合，每一位对应一个信号。

- **mask**：信号掩码，用于屏蔽某些信号。被屏蔽的信号不会被处理，直到掩码被清除。

- **handling**：记录当前正在处理的信号状态。如果进程正在处理信号，新收到的信号可能会被延迟处理。

- **actions**：信号处理函数数组，每个信号可以注册一个处理函数。如果某个信号没有注册处理函数，则执行默认行为。

HandlingSignal 枚举
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   pub enum HandlingSignal {
       Frozen,                   // 内核信号，进程被暂停
       UserSignal(LocalContext), // 用户信号，保存了原来的用户上下文
   }

- **Frozen**：表示进程因为收到 SIGSTOP 等内核信号而被暂停。当收到 SIGCONT 信号时，进程会恢复执行。

- **UserSignal**：表示进程正在处理用户注册的信号处理函数。此时需要保存原来的用户上下文，以便处理函数返回后恢复执行。

SignalSet 数据结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``SignalSet`` 是一个位集合，用于高效地存储和操作信号集合，位于 ``rCore-Tutorial-in-single-workspace/signal-impl/src/signal_set.rs``：

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/signal_set.rs

   #[derive(Clone, Copy, Debug)]
   pub struct SignalSet(pub usize);

   impl SignalSet {
       pub fn empty() -> Self;
       pub fn contain_bit(&self, kth: usize) -> bool;
       pub fn add_bit(&mut self, kth: usize);
       pub fn remove_bit(&mut self, kth: usize);
       pub fn find_first_one(&self, mask: SignalSet) -> Option<usize>;
       // ...
   }

``SignalSet`` 使用一个 ``usize`` 值作为位掩码，每一位对应一个信号。这种设计使得信号集合的操作（如添加、删除、查找）都可以在常数时间内完成，非常高效。

小结
--------------------------------------------

本章介绍了信号机制的基本概念和架构设计。信号是 Unix/Linux 系统中一种重要的进程间通信机制，允许进程之间发送异步通知。我们将信号机制的实现分为三个模块：``signal-defs`` 定义基础数据结构，``signal`` 定义接口，``signal-impl`` 提供具体实现。这种设计提高了代码的模块化和可扩展性。

下一节我们将深入信号机制的实现细节，了解信号是如何被存储、管理和处理的。
