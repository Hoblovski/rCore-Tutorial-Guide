信号相关系统调用
============================================

本章实现了四个与信号相关的系统调用：``sys_kill``、``sys_sigaction``、``sys_sigprocmask`` 和 ``sys_sigreturn``。这些系统调用允许用户程序发送信号、注册信号处理函数、设置信号掩码，以及从信号处理函数返回。

sys_kill - 发送信号
--------------------------------------------

系统调用原型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust

   /// 功能：向指定进程发送信号。
   /// 参数：pid 表示目标进程的进程 ID，signum 表示要发送的信号编号。
   /// 返回值：如果成功则返回 0，否则返回 -1。
   /// syscall ID：根据 syscall 模块定义
   pub fn sys_kill(pid: isize, signum: u8) -> isize;

``sys_kill`` 系统调用允许一个进程向另一个进程（或自己）发送信号。这是信号机制中最基本的操作。

实现细节
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust
   :linenos:
   :caption: ch7/src/main.rs

   impl Signal for SyscallContext {
       fn kill(&self, _caller: Caller, pid: isize, signum: u8) -> isize {
           if let Some(target_task) =
               unsafe { PROCESSOR.get_task(ProcId::from_usize(pid as usize)) }
           {
               if let Ok(signal_no) = SignalNo::try_from(signum) {
                   if signal_no != SignalNo::ERR {
                       target_task.signal.add_signal(signal_no);
                       return 0;
                   }
               }
           }
           -1
       }
   }

实现流程：

1. **查找目标进程**：根据进程 ID 在进程管理器中查找目标进程。如果进程不存在，返回 -1。

2. **验证信号编号**：将 ``signum`` 转换为 ``SignalNo`` 枚举。如果信号编号无效（转换为 ``ERR``），返回 -1。

3. **添加信号**：调用目标进程信号模块的 ``add_signal()`` 方法，将信号添加到目标进程的信号队列中。

4. **返回结果**：如果成功，返回 0；否则返回 -1。

使用示例
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下是一个简单的使用示例，进程向自己发送 SIGUSR1 信号：

.. code-block:: rust
   :linenos:
   :caption: user/src/bin/sig_simple.rs

   #[no_mangle]
   pub extern "C" fn main() -> i32 {
       // 注册信号处理函数
       let mut new = SignalAction::default();
       new.handler = func as usize;
       sigaction(SignalNo::SIGUSR1, &new, &mut SignalAction::default());
       
       // 向自己发送信号
       kill(getpid(), SignalNo::SIGUSR1);
       
       0
   }

sys_sigaction - 设置信号处理函数
--------------------------------------------

系统调用原型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust

   /// 功能：设置或获取信号处理函数。
   /// 参数：signum 表示信号编号，action 表示新的信号处理函数（如果为 NULL 则不设置），
   ///       old_action 表示用于返回旧的信号处理函数（如果为 NULL 则不返回）。
   /// 返回值：如果成功则返回 0，否则返回 -1。
   /// syscall ID：根据 syscall 模块定义
   pub fn sys_sigaction(signum: u8, action: usize, old_action: usize) -> isize;

``sys_sigaction`` 系统调用允许进程注册信号处理函数，或获取当前已注册的处理函数。这是信号机制的核心功能之一。

实现细节
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust
   :linenos:
   :caption: ch7/src/main.rs

   fn sigaction(
       &self,
       _caller: Caller,
       signum: u8,
       action: usize,
       old_action: usize,
   ) -> isize {
       if signum as usize > signal::MAX_SIG {
           return -1;
       }
       let current = unsafe { PROCESSOR.current().unwrap() };
       if let Ok(signal_no) = SignalNo::try_from(signum) {
           if signal_no == SignalNo::ERR {
               return -1;
           }
           // 如果需要返回原来的处理函数
           if old_action as usize != 0 {
               if let Some(mut ptr) = current
                   .address_space
                   .translate(VAddr::new(old_action), WRITEABLE)
               {
                   if let Some(signal_action) = current.signal.get_action_ref(signal_no) {
                       *unsafe { ptr.as_mut() } = signal_action;
                   } else {
                       return -1;
                   }
               } else {
                   return -1;
               }
           }
           // 如果需要设置新的处理函数
           if action as usize != 0 {
               if let Some(ptr) = current
                   .address_space
                   .translate(VAddr::new(action), READABLE)
               {
                   if !current
                       .signal
                       .set_action(signal_no, &unsafe { *ptr.as_ptr() })
                   {
                       return -1;
                   }
               } else {
                   return -1;
               }
           }
           return 0;
       }
       -1
   }

实现流程：

1. **验证信号编号**：检查 ``signum`` 是否在有效范围内（不超过 ``MAX_SIG``），并转换为 ``SignalNo`` 枚举。如果无效，返回 -1。

2. **返回旧的处理函数**：如果 ``old_action`` 不为 NULL，则从信号模块中获取当前的处理函数，并将其写入用户地址空间。如果获取失败或地址无效，返回 -1。

3. **设置新的处理函数**：如果 ``action`` 不为 NULL，则从用户地址空间读取新的处理函数，并调用 ``set_action()`` 方法设置。注意，不能为 SIGKILL 和 SIGSTOP 设置处理函数，``set_action()`` 会返回 false，此时返回 -1。

4. **返回结果**：如果成功，返回 0；否则返回 -1。

使用示例
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下示例展示了如何注册 SIGINT 信号的处理函数：

.. code-block:: rust
   :linenos:
   :caption: user/src/bin/sig_ctrlc.rs

   fn func() {
       println!("signal_handler: caught signal SIGINT, and exit(1)");
       exit(1);
   }

   #[no_mangle]
   pub extern "C" fn main() -> i32 {
       let mut new = SignalAction::default();
       let old = SignalAction::default();
       new.handler = func as usize;  // 设置处理函数地址
       
       // 注册信号处理函数
       if sigaction(SignalNo::SIGINT, &new, &old) < 0 {
           panic!("Sigaction failed!");
       }
       
       // 等待用户输入（可能会收到 SIGINT 信号）
       loop {
           let c = getchar();
           if c == LF || c == CR {
               break;
           }
       }
       
       0
   }

当用户按下 Ctrl+C 时，内核会向进程发送 SIGINT 信号，进程注册的处理函数 ``func`` 会被调用。

sys_sigprocmask - 设置信号掩码
--------------------------------------------

系统调用原型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust

   /// 功能：设置信号掩码。
   /// 参数：mask 表示新的信号掩码。
   /// 返回值：返回旧的信号掩码。
   /// syscall ID：根据 syscall 模块定义
   pub fn sys_sigprocmask(mask: usize) -> isize;

``sys_sigprocmask`` 系统调用允许进程设置信号掩码，屏蔽某些信号。被屏蔽的信号不会被处理，直到掩码被清除。

实现细节
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust
   :linenos:
   :caption: ch7/src/main.rs

   fn sigprocmask(&self, _caller: Caller, mask: usize) -> isize {
       let current = unsafe { PROCESSOR.current().unwrap() };
       current.signal.update_mask(mask) as isize
   }

实现非常简单：

1. **获取当前进程**：从进程管理器中获取当前进程。

2. **更新信号掩码**：调用信号模块的 ``update_mask()`` 方法，设置新的信号掩码，并返回旧的掩码。

3. **返回旧掩码**：将旧的信号掩码作为返回值返回。

信号掩码是一个位掩码，每一位对应一个信号。如果某一位为 1，表示该信号被屏蔽；如果为 0，表示该信号未被屏蔽。

使用示例
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下示例展示了如何使用信号掩码屏蔽 SIGUSR1 信号：

.. code-block:: rust

   // 创建一个信号掩码，屏蔽 SIGUSR1（信号编号为 10）
   let mask = 1 << 10;
   
   // 设置信号掩码，保存旧的掩码
   let old_mask = sigprocmask(mask);
   
   // 此时 SIGUSR1 信号会被屏蔽，不会被处理
   // ...
   
   // 恢复旧的掩码
   sigprocmask(old_mask);

在关键代码段执行期间，可以临时屏蔽某些信号，保证代码的原子性。执行完毕后，再恢复原来的掩码。

sys_sigreturn - 从信号处理函数返回
--------------------------------------------

系统调用原型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust

   /// 功能：从信号处理函数返回，恢复原来的用户上下文。
   /// 参数：无。
   /// 返回值：如果成功则返回 0，否则返回 -1。
   /// syscall ID：根据 syscall 模块定义
   pub fn sys_sigreturn() -> isize;

``sys_sigreturn`` 系统调用用于从信号处理函数返回，恢复被信号中断的代码的执行。这是信号处理流程的最后一步。

实现细节
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust
   :linenos:
   :caption: ch7/src/main.rs

   fn sigreturn(&self, _caller: Caller) -> isize {
       let current = unsafe { PROCESSOR.current().unwrap() };
       // 如成功，则需要修改当前用户程序的 LocalContext
       if current.signal.sig_return(&mut current.context.context) {
           0
       } else {
           -1
       }
   }

实现流程：

1. **获取当前进程**：从进程管理器中获取当前进程。

2. **恢复上下文**：调用信号模块的 ``sig_return()`` 方法，恢复被信号中断的代码的上下文。该方法会从 ``handling`` 字段中取出保存的上下文，并将其恢复到 ``current_context`` 中。

3. **返回结果**：如果成功恢复上下文，返回 0；否则返回 -1。

使用示例
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

信号处理函数在执行完毕后，必须调用 ``sigreturn`` 系统调用：

.. code-block:: rust
   :linenos:
   :caption: user/src/bin/sig_simple.rs

   fn func() {
       println!("user_sig_test succsess");
       sigreturn();  // 必须调用 sigreturn 返回
   }

   #[no_mangle]
   pub extern "C" fn main() -> i32 {
       let mut new = SignalAction::default();
       new.handler = func as usize;
       sigaction(SignalNo::SIGUSR1, &new, &mut SignalAction::default());
       
       kill(getpid(), SignalNo::SIGUSR1);
       
       // 信号处理函数执行完毕后，会继续执行这里的代码
       println!("signal_simple: Done");
       0
   }

注意，信号处理函数必须调用 ``sigreturn`` 系统调用，否则进程无法恢复原来的执行上下文，可能导致不可预期的行为。

信号处理在内核中的集成
--------------------------------------------

在内核的主循环中，我们会在系统调用处理之后检查并处理信号：

.. code-block:: rust
   :linenos:
   :caption: ch7/src/main.rs

   match scause::read().cause() {
       scause::Trap::Exception(scause::Exception::UserEnvCall) => {
           // 处理系统调用
           let syscall_ret = syscall::handle(Caller { entity: 0, flow: 0 }, id, args);
           
           // 处理信号
           match task.signal.handle_signals(ctx) {
               SignalResult::ProcessKilled(exit_code) => unsafe {
                   PROCESSOR.make_current_exited(exit_code as _)
               },
               _ => match syscall_ret {
                   Ret::Done(ret) => match id {
                       Id::EXIT => unsafe { PROCESSOR.make_current_exited(ret) },
                       _ => {
                           let ctx = &mut task.context.context;
                           *ctx.a_mut(0) = ret as _;
                           unsafe { PROCESSOR.make_current_suspend() };
                       }
                   },
                   // ...
               },
           }
       }
       // ...
   }

信号处理的时机是在系统调用返回之前。这样设计的好处是：

1. **原子性**：系统调用和信号处理都在内核态完成，保证了操作的原子性。

2. **及时性**：系统调用是进程从用户态进入内核态的主要途径，在这个时机检查信号可以及时响应。

3. **安全性**：在内核态处理信号可以避免用户程序的干扰，保证信号处理的正确性。

.. note::

   当前实现将信号处理放在系统调用之后，这只是临时的实现。正确的处理位置应该是在 trap 中处理异常和中断之后，返回用户态之前。例如，当发现访存异常时，应该触发 SIGSEGV 信号然后进行处理。但目前 syscall 之后直接切换用户程序，没有"返回用户态"这一步，甚至 trap 本身也没了。处理信号的具体时机还需要后续再讨论。

小结
--------------------------------------------

本章介绍了四个与信号相关的系统调用：``sys_kill`` 用于发送信号，``sys_sigaction`` 用于注册信号处理函数，``sys_sigprocmask`` 用于设置信号掩码，``sys_sigreturn`` 用于从信号处理函数返回。这些系统调用共同构成了信号机制的完整功能。

信号机制是 Unix/Linux 系统中一种重要的进程间通信机制，它允许进程之间发送异步通知，实现进程控制、异常处理、用户交互等功能。通过本章的学习，我们了解了信号机制的完整实现，包括信号的发送、接收、处理和返回等各个环节。
