信号实现细节
============================================

信号的存储和管理
--------------------------------------------

在 ``SignalImpl`` 中，信号的状态通过以下几个字段管理：

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

- **received**：使用 ``SignalSet`` 存储已收到但尚未处理的信号。当进程收到信号时，信号会被添加到 ``received`` 中。``SignalSet`` 是一个位集合，每一位对应一个信号，可以高效地进行添加、删除和查找操作。

- **mask**：信号掩码，用于屏蔽某些信号。被屏蔽的信号不会被处理，即使它们已经在 ``received`` 中。信号掩码的主要作用是：
  - 在信号处理函数执行期间，屏蔽某些信号，防止处理函数被中断
  - 在关键代码段执行期间，临时屏蔽某些信号，保证代码的原子性

- **handling**：记录当前正在处理的信号状态。如果进程正在处理信号，新收到的信号可能会被延迟处理，直到当前信号处理完成。

- **actions**：信号处理函数数组，每个信号可以注册一个处理函数。如果某个信号没有注册处理函数（``actions[signum]`` 为 ``None``），则执行默认行为。

信号处理流程
--------------------------------------------

信号处理的核心是 ``handle_signals()`` 方法。当进程从内核态返回用户态时（如系统调用返回），内核会调用此方法检查并处理待处理的信号。

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   fn handle_signals(&mut self, current_context: &mut LocalContext) -> SignalResult {
       // 如果正在处理信号
       if self.is_handling_signal() {
           match self.handling.as_ref().unwrap() {
               // 如果当前正在暂停状态
               HandlingSignal::Frozen => {
                   // 检查是否收到 SIGCONT，如果收到则恢复执行
                   if self.fetch_and_remove(SignalNo::SIGCONT) {
                       self.handling.take();
                       SignalResult::Handled
                   } else {
                       // 否则，继续暂停
                       SignalResult::ProcessSuspended
                   }
               }
               // 其他情况下，需要等待当前信号处理结束
               _ => SignalResult::IsHandlingSignal,
           }
       } else if let Some(signal) = self.fetch_signal() {
           // 从信号队列中取出一个未被屏蔽的信号
           match signal {
               // SIGKILL 信号不能被捕获或忽略
               SignalNo::SIGKILL => SignalResult::ProcessKilled(-(signal as i32)),
               // SIGSTOP 信号暂停进程
               SignalNo::SIGSTOP => {
                   self.handling = Some(HandlingSignal::Frozen);
                   SignalResult::ProcessSuspended
               }
               _ => {
                   if let Some(action) = self.actions[signal as usize] {
                       // 如果用户给定了处理方式，则按照 SignalAction 中的描述处理
                       // 保存原来用户程序的上下文信息
                       self.handling = Some(HandlingSignal::UserSignal(current_context.clone()));
                       // 修改返回后的 pc 值为 handler，修改 a0 为信号编号
                       *current_context.pc_mut() = action.handler;
                       *current_context.a_mut(0) = signal as usize;
                       SignalResult::Handled
                   } else {
                       // 否则，使用默认行为处理
                       DefaultAction::from(signal).into()
                   }
               }
           }
       } else {
           SignalResult::NoSignal
       }
   }

信号处理的流程如下：

1. **检查是否正在处理信号**：如果 ``handling`` 不为 ``None``，说明进程正在处理信号。此时需要根据处理状态决定下一步操作：
   - 如果是 ``Frozen`` 状态（进程被暂停），检查是否收到 SIGCONT 信号。如果收到，则恢复进程执行；否则继续暂停。
   - 如果是 ``UserSignal`` 状态（正在执行用户处理函数），则忽略新收到的信号，等待当前处理完成。

2. **获取待处理的信号**：调用 ``fetch_signal()`` 方法从 ``received`` 中取出一个未被 ``mask`` 屏蔽的信号。如果没有这样的信号，返回 ``NoSignal``。

3. **处理特殊信号**：
   - **SIGKILL**：不能被捕获或忽略，直接终止进程。
   - **SIGSTOP**：暂停进程，将 ``handling`` 设置为 ``Frozen``。

4. **调用用户处理函数或执行默认行为**：
   - 如果信号注册了处理函数（``actions[signal]`` 不为 ``None``），则：
     - 保存当前用户上下文到 ``handling`` 中
     - 修改程序计数器（PC）指向处理函数地址
     - 将信号编号作为参数传递给处理函数（通过 a0 寄存器）
     - 返回 ``Handled``，内核会直接返回用户态执行处理函数
   - 如果没有注册处理函数，则执行默认行为（通常是终止进程或忽略信号）。

信号处理函数的调用机制
--------------------------------------------

当进程收到信号并需要调用用户注册的处理函数时，内核需要：

1. **保存当前上下文**：将当前的用户程序上下文（包括所有寄存器、栈指针等）保存到 ``handling`` 字段中，以便处理函数返回后恢复执行。

2. **修改程序计数器**：将程序计数器（PC）设置为信号处理函数的地址（``action.handler``），使得进程返回用户态时直接执行处理函数。

3. **传递参数**：将信号编号作为参数传递给处理函数。在 RISC-V 架构中，第一个参数通过 a0 寄存器传递，因此将信号编号写入 a0 寄存器。

4. **设置信号掩码**：在处理函数执行期间，应用 ``action.mask`` 中指定的信号掩码，屏蔽某些信号，防止处理函数被中断。

当处理函数执行完毕后，进程会调用 ``sigreturn`` 系统调用，内核会调用 ``sig_return()`` 方法恢复原来的上下文：

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   fn sig_return(&mut self, current_context: &mut LocalContext) -> bool {
       let handling_signal = self.handling.take();
       match handling_signal {
           Some(HandlingSignal::UserSignal(old_ctx)) => {
               // 恢复原来的用户上下文
               *current_context = old_ctx;
               true
           }
           // 如果当前在处理内核信号，或者没有在处理信号，也就谈不上"返回"了
           _ => {
               self.handling = handling_signal;
               false
           }
       }
   }

``sig_return()`` 方法会从 ``handling`` 中取出保存的上下文，并将其恢复到 ``current_context`` 中。这样，进程就可以继续执行被信号中断的代码。

特殊信号的处理
--------------------------------------------

某些信号具有特殊的语义，不能被用户程序捕获或忽略：

SIGKILL 和 SIGSTOP
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``SIGKILL`` 和 ``SIGSTOP`` 是两个特殊的信号，它们不能被捕获、阻塞或忽略：

- **SIGKILL**：强制终止进程。无论进程处于什么状态，收到 SIGKILL 信号后都会被立即终止。这是终止进程的最可靠方式，常用于处理"僵尸进程"或无法正常退出的进程。

- **SIGSTOP**：暂停进程的执行。进程收到 SIGSTOP 信号后会被立即暂停，无法继续执行，直到收到 SIGCONT 信号。

在 ``set_action()`` 方法中，我们禁止为这两个信号设置处理函数：

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   fn set_action(&mut self, signum: SignalNo, action: &SignalAction) -> bool {
       if signum == SignalNo::SIGKILL || signum == SignalNo::SIGSTOP {
           false  // 不能为 SIGKILL 和 SIGSTOP 设置处理函数
       } else {
           self.actions[signum as usize] = Some(*action);
           true
       }
   }

SIGCONT
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``SIGCONT`` 用于恢复被 SIGSTOP 暂停的进程。当进程处于 ``Frozen`` 状态时，如果收到 SIGCONT 信号，进程会恢复执行：

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   HandlingSignal::Frozen => {
       // 检查是否收到 SIGCONT，如果收到则恢复执行
       if self.fetch_and_remove(SignalNo::SIGCONT) {
           self.handling.take();
           SignalResult::Handled
       } else {
           // 否则，继续暂停
           SignalResult::ProcessSuspended
       }
   }

默认行为
--------------------------------------------

当信号没有注册处理函数时，系统会执行默认行为。默认行为由 ``DefaultAction`` 枚举定义，位于 ``rCore-Tutorial-in-single-workspace/signal-impl/src/default_action.rs``：

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/default_action.rs

   pub enum DefaultAction {
       Terminate(i32),  // 结束进程，返回退出码
       Ignore,          // 忽略信号
   }

   impl From<SignalNo> for DefaultAction {
       fn from(signal_no: SignalNo) -> Self {
           match signal_no {
               SignalNo::SIGCHLD | SignalNo::SIGURG => Self::Ignore,
               _ => Self::Terminate(-(signal_no as i32)),
           }
       }
   }

大多数信号的默认行为是终止进程，退出码为信号编号的负值。但有些信号（如 SIGCHLD、SIGURG）的默认行为是忽略，这些信号通常用于通知进程某个事件已经发生，但不要求进程立即响应。

信号掩码的屏蔽机制
--------------------------------------------

信号掩码用于屏蔽某些信号，防止它们在特定时刻被处理。信号掩码的主要用途包括：

1. **在信号处理函数执行期间屏蔽信号**：当进程正在执行信号处理函数时，可能需要屏蔽某些信号，防止处理函数被中断。这可以通过 ``SignalAction.mask`` 字段实现。

2. **在关键代码段执行期间屏蔽信号**：当进程执行关键代码段时，可能需要临时屏蔽某些信号，保证代码的原子性。这可以通过 ``sigprocmask`` 系统调用实现。

在 ``fetch_signal()`` 方法中，我们只返回未被掩码屏蔽的信号：

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   fn fetch_signal(&mut self) -> Option<SignalNo> {
       // 在已收到的信号中，寻找一个没有被 mask 屏蔽的信号
       self.received.find_first_one(self.mask).map(|num| {
           self.received.remove_bit(num);
           num.into()
       })
   }

``find_first_one()`` 方法会在 ``received`` 中查找第一个不在 ``mask`` 中的信号。如果找到，则从 ``received`` 中移除该信号并返回；如果没有找到，则返回 ``None``。

fork 和 exec 时的信号行为
--------------------------------------------

信号在 ``fork`` 和 ``exec`` 时的行为是不同的：

fork 时的信号继承
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当进程调用 ``fork`` 创建子进程时，子进程会继承父进程的信号处理函数和掩码：

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   fn from_fork(&mut self) -> Box<dyn Signal> {
       Box::new(Self {
           received: SignalSet::empty(),  // 子进程的信号队列为空
           mask: self.mask,               // 继承父进程的信号掩码
           handling: None,                // 子进程没有正在处理的信号
           actions: {
               // 继承父进程的信号处理函数
               let mut actions = [None; MAX_SIG + 1];
               actions.copy_from_slice(&self.actions);
               actions
           },
       })
   }

注意，子进程的信号队列（``received``）是空的，因为子进程不应该继承父进程已收到但尚未处理的信号。但子进程会继承父进程的信号处理函数和掩码，因为它们是进程的配置信息，应该保持一致。

exec 时的信号清除
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当进程调用 ``exec`` 执行新程序时，需要清除所有信号处理函数（但保留信号掩码）：

.. code-block:: rust
   :linenos:
   :caption: signal-impl/src/lib.rs

   fn clear(&mut self) {
       for action in &mut self.actions {
           action.take();  // 清除所有信号处理函数
       }
   }

这是因为新程序可能有自己的信号处理逻辑，不应该继承旧程序的处理函数。但信号掩码会被保留，因为它是进程的全局配置，不应该因为执行新程序而改变。

在进程结构中的集成
--------------------------------------------

在 ``Process`` 结构体中，我们添加了 ``signal: Box<dyn Signal>`` 字段：

.. code-block:: rust
   :linenos:
   :caption: ch7/src/process.rs

   pub struct Process {
       pub pid: ProcId,
       pub context: ForeignContext,
       pub address_space: AddressSpace<Sv39, Sv39Manager>,
       pub fd_table: Vec<Option<Mutex<FileHandle>>>,
       /// 信号模块
       pub signal: Box<dyn Signal>,
   }

在创建新进程时，我们初始化信号模块：

.. code-block:: rust
   :linenos:
   :caption: ch7/src/process.rs

   pub fn from_elf(elf: ElfFile) -> Option<Self> {
       // ...
       Some(Self {
           pid: ProcId::new(),
           context: ForeignContext { context, satp },
           address_space,
           fd_table: vec![
               Some(Mutex::new(FileHandle::empty(true, false))),   // Stdin
               Some(Mutex::new(FileHandle::empty(false, true))),   // Stdout
           ],
           signal: Box::new(SignalImpl::new()),  // 初始化信号模块
       })
   }

在 ``fork`` 时，我们调用 ``from_fork()`` 方法创建子进程的信号模块：

.. code-block:: rust
   :linenos:
   :caption: ch7/src/process.rs

   pub fn fork(&mut self) -> Option<Process> {
       // ...
       Some(Self {
           pid,
           context: foreign_ctx,
           address_space,
           fd_table: new_fd_table,
           signal: self.signal.from_fork(),  // 继承父进程的信号处理
       })
   }

在 ``exec`` 时，我们调用 ``clear()`` 方法清除信号处理函数：

.. code-block:: rust
   :linenos:
   :caption: ch7/src/process.rs

   pub fn exec(&mut self, elf: ElfFile) {
       let proc = Process::from_elf(elf).unwrap();
       self.address_space = proc.address_space;
       self.context = proc.context;
       self.signal.clear();  // 清除信号处理函数
   }

小结
--------------------------------------------

本章深入介绍了信号机制的实现细节。信号的处理流程包括：从信号队列中取出信号、检查信号类型、调用处理函数或执行默认行为。信号处理函数通过保存和恢复用户上下文来实现，使得进程可以在处理完信号后继续执行被中断的代码。特殊信号（如 SIGKILL、SIGSTOP）具有特殊的语义，不能被用户程序捕获或忽略。信号在 ``fork`` 时会继承处理函数和掩码，在 ``exec`` 时会清除处理函数但保留掩码。

下一节我们将介绍信号相关的系统调用，了解用户程序如何使用信号机制。
