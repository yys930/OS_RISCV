# Lab4：进程管理
## 2213781 宋宣昊  2210554 刘志威   2212850 王海鹏

### 练习1：分配并初始化一个进程控制块（需要编码）
alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。
> 【提示】在alloc_proc函数的实现中，需要初始化的proc_struct结构中的成员变量至少包括：state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name。
请在实验报告中简要说明你的设计实现过程。请回答如下问题：
- 请说明proc_struct中`struct context context`和`struct trapframe *tf`成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）

#### 设计实现过程
补充代码如下所示：
```cpp
        proc->state = PROC_UNINIT;               // 进程状态初始化为PROC_UNINIT
        proc->pid = -1;                          // PID初始化为-1
        proc->runs = 0;                          // 初始运行次数为0
        proc->kstack = 0;                        // 内核栈地址初始化为0
        proc->need_resched = 0;                  // 默认不需要调度
        proc->parent = NULL;                     // 父进程初始化为NULL
        proc->mm = NULL;                         // 内存管理结构初始化为NULL
        memset(&proc->context, 0, sizeof(struct context));  // 清空上下文
        proc->tf = NULL;                         // Trap frame初始化为NULL
        proc->cr3 = boot_cr3;                           // 页目录基地址初始化为boot_cr3
        proc->flags = 0;                         // 标志初始化为0
        memset(proc->name, 0, sizeof(proc->name)); // 进程名清空
```
其中主要初始化值参考下面的代码：
```cpp
if(idleproc->cr3 == boot_cr3 && idleproc->tf == NULL && !context_init_flag
        && idleproc->state == PROC_UNINIT && idleproc->pid == -1 && idleproc->runs == 0
        && idleproc->kstack == 0 && idleproc->need_resched == 0 && idleproc->parent == NULL
        && idleproc->mm == NULL && idleproc->flags == 0 && !proc_name_flag
    ){
        cprintf("alloc_proc() correct!\n");

    }
```
#### 问题回答
##### **`struct context context`：**
- **含义**：用于保存进程在上下文切换时的 CPU 寄存器状态，包括栈指针（SP）以及其他通用寄存器。
- **作用**：
  - **上下文切换**：当调度器决定切换当前运行的进程时，会保存当前进程的 `context`，然后加载下一个待运行进程的 `context`，从而实现进程间的切换。
  - **进程恢复**：确保每个进程在被调度回来时，能够从上次中断的地方继续执行，保持进程执行的连续性。
- **成员变量**
```cpp
struct context {
    uintptr_t ra;
    uintptr_t sp;
    uintptr_t s0;
    uintptr_t s1;
    uintptr_t s2;
    uintptr_t s3;
    uintptr_t s4;
    uintptr_t s5;
    uintptr_t s6;
    uintptr_t s7;
    uintptr_t s8;
    uintptr_t s9;
    uintptr_t s10;
    uintptr_t s11;
};
```
1. `uintptr_t ra;` （返回地址寄存器）

- **含义**：
  - `ra` 代表 **Return Address**（返回地址）寄存器。在 RISC-V 架构中，`ra` 是寄存器 x1，用于存储函数调用的返回地址。
  
- **作用**：
  - **函数调用与返回**：在函数调用时，调用者的返回地址被保存到 `ra` 寄存器。函数执行完毕后，通过 `ra` 寄存器返回到调用者的指令位置。
  - **上下文切换**：在进行上下文切换时，当前进程的 `ra` 值被保存，以便在恢复该进程时能够正确返回到先前的执行位置。

2. `uintptr_t sp;` （栈指针寄存器）

- **含义**：
  - `sp` 代表 **Stack Pointer**（栈指针）寄存器。在 RISC-V 架构中，`sp` 是寄存器 x2，用于指向当前栈的顶端。
  
- **作用**：
  - **栈管理**：`sp` 指向当前栈帧的顶端，用于管理函数调用中的局部变量、返回地址等。
  - **上下文切换**：在切换到另一个进程时，当前进程的 `sp` 被保存，以便在恢复该进程时能够恢复其栈状态。

3. `uintptr_t s0;` 至 `uintptr_t s11;` （保存寄存器）

- **含义**：
  - `s0` 到 `s11` 代表 **Saved Registers**（保存寄存器），对应 RISC-V 架构中的寄存器 x8 到 x18。这些寄存器用于保存临时数据或函数调用中的保存值。
  
- **作用**：
  - **数据保存**：这些寄存器用于保存函数调用中的临时数据，避免在函数调用过程中被破坏。
  - **上下文切换**：在上下文切换时，这些寄存器的值被保存到 `struct context` 中，以便在恢复进程时能够恢复其执行状态。


##### **`struct trapframe *tf`：**
- **含义**：用于保存进程在发生陷阱（如中断、系统调用、异常）时的 CPU 寄存器状态，包括程序计数器（EPC）、状态寄存器（Status）以及其他相关寄存器。
- **作用**：
  - **陷阱处理**：当进程发生陷阱时，硬件会将当前的寄存器状态保存到 `trapframe` 中，操作系统通过处理这些寄存器值来响应陷阱。
  - **状态恢复**：在陷阱处理完成后，操作系统可以通过 `trapframe` 恢复进程的寄存器状态，使进程能够继续执行。
- **成员变量**：
```cpp
struct trapframe {
    struct pushregs gpr;
    uintptr_t status;
    uintptr_t epc;
    uintptr_t tval;
    uintptr_t cause;
};
```
1. **`struct pushregs gpr;`**

   - **含义**：
     `gpr` 是一个结构体，通常用于保存**通用寄存器（General-Purpose Registers）**的值。在不同的架构中，通用寄存器的数量和名称可能有所不同，但它们一般用于存储临时数据、函数参数、返回值等。
   
   - **作用**：
     - **保存寄存器状态**：当发生陷阱时，当前进程的通用寄存器状态会被保存到 `gpr` 中，以便在陷阱处理完成后能够恢复到陷阱发生前的状态。
     - **支持上下文切换**：在多任务操作系统中，保存和恢复寄存器状态是上下文切换的关键步骤。`gpr` 结构体提供了一种有组织的方式来存储这些寄存器值。


2. **`uintptr_t status;`**

   - **含义**：
     `status` 通常用于保存**状态寄存器（Status Register）**的值。状态寄存器包含了当前处理器的状态信息，如中断使能位、当前运行模式（用户态或内核态）、异常级别等。

   - **作用**：
     - **保存处理器状态**：在陷阱发生时，处理器的状态寄存器值被保存到 `status` 中，以便在陷阱处理完成后能够恢复到陷阱发生前的处理器状态。
     - **控制陷阱处理行为**：通过修改 `status`，操作系统可以控制陷阱处理期间的行为，例如临时禁用中断，切换处理器模式等。


3. **`uintptr_t epc;`**

   - **含义**：
     `epc` 代表**异常程序计数器（Exception Program Counter）**，即**发生陷阱时的程序计数器（PC）**的值。它指示了陷阱发生时，处理器正在执行的指令地址。

   - **作用**：
     - **记录陷阱发生位置**：`epc` 保存了陷阱发生时的指令地址，使得在陷阱处理完成后，操作系统可以将程序计数器恢复到该位置，继续执行被中断的程序。
     - **支持异常处理**：在处理某些异常（如页错误、非法指令等）时，操作系统可能需要根据 `epc` 的值决定如何处理异常。


4. **`uintptr_t tval;`**

   - **含义**：
     `tval` 通常用于保存**陷阱值寄存器（Trap Value Register）**的值。它包含了与陷阱相关的具体信息，如触发陷阱的原因、错误码或其他相关数据。

   - **作用**：
     - **提供陷阱详情**：`tval` 保存了导致陷阱发生的具体值或错误码，帮助操作系统确定陷阱的原因并采取相应的处理措施。
     - **支持异常诊断**：在调试或处理异常时，`tval` 提供了有用的信息，可以用于诊断问题所在。


5. **`uintptr_t cause;`**

   - **含义**：
     `cause` 用于保存**陷阱原因寄存器（Cause Register）**的值。它指示了导致陷阱发生的具体原因，如中断类型、异常类型、系统调用等。

   - **作用**：
     - **识别陷阱类型**：通过 `cause`，操作系统可以确定陷阱是由于中断、异常还是系统调用引起的，从而采取不同的处理策略。
     - **支持多种陷阱处理**：不同类型的陷阱需要不同的处理逻辑，`cause` 提供了区分这些陷阱类型的依据。



##### **在本实验中的具体作用：**
- **`context`**：
  - 在 `alloc_proc` 中初始化为零，确保新进程在首次调度时拥有一个干净的上下文。
  - 在 `copy_thread` 函数中设置 `context.ra` 为 `forkret`，确保新进程在切换到内核态后能够执行 `forkret` 函数，从而开始其执行。

- **`tf`**：
  - 在 `copy_thread` 中分配并初始化 `trapframe`，将父进程的 `trapframe` 复制到子进程，并根据需要修改特定寄存器（如 `a0`）以区分父子进程。
  - 确保新进程在处理系统调用或陷阱时，能够正确恢复其执行状态。
##### **总结**
 `struct context` 和 `struct trapframe *tf` 确保了 Ucore 操作系统能够有效地管理进程的上下文切换和陷阱处理。这对于实现多任务和高效的进程管理至关重要。


### 练习2：为新创建的内核线程分配资源（需要编码）
创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。因此，我们实际需要"fork"的东西就是stack和trapframe。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：
- 调用alloc_proc，首先获得一块用户信息块。
- 为进程分配一个内核栈。
- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
- 复制原进程上下文到新进程
- 将新进程添加到进程列表
- 唤醒新进程
- 返回新进程号

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

- 填写的do_fork函数代码如下：
#### 1. 分配新的进程结构
 
```c
proc = alloc_proc();
if (proc == NULL) {
    goto bad_fork_cleanup_proc;
}
proc->parent = current;
```
- alloc_proc()：这是一个函数调用，用于从内核的内存池中分配一个新的进程控制块（Process Control Block, PCB）。PCB是操作系统用来存储进程状态、寄存器内容、内存管理信息等的数据结构。

- 错误处理：如果alloc_proc()返回NULL，表示内存不足，无法分配新的PCB。此时，代码会跳转到bad_fork_cleanup_proc标签进行错误处理，通常这意味着释放已分配的资源并退出函数。

- 设置父进程：proc->parent = current;将新进程的父进程设置为当前进程（current）。

#### 2. 为子进程分配内核栈
```c
if (setup_kstack(proc) != 0) {
    goto bad_fork_cleanup_proc;
}
```
- setup_kstack(proc)：这是一个函数调用，用于为子进程分配一个独立的内核栈。内核栈是进程在内核模式下执行时所使用的栈空间。
- 错误处理：如果setup_kstack()返回非0值，表示分配内核栈失败。此时，代码会跳转到bad_fork_cleanup_proc标签进行错误处理。

#### 3. 判断并复制内存管理信息
```c
if (copy_mm(clone_flags, proc) != 0) {
    goto bad_fork_cleanup_kstack;
}
```
- copy_mm(clone_flags, proc)：这是一个函数调用，用于根据clone_flags标志判断是否需要复制当前进程的内存管理信息到子进程中。clone_flags决定了是共享内存空间还是复制一份新的内存空间。
- 错误处理：如果copy_mm()返回非0值，表示复制内存管理信息失败。此时，代码会跳转到bad_fork_cleanup_kstack标签进行错误处理，通常会释放已分配的内核栈。

#### 4. 设置进程的执行上下文
```c
copy_thread(proc, stack, tf);
```
- copy_thread(proc, stack, tf)：这是一个函数调用，用于设置子进程的执行上下文，包括初始化寄存器、栈指针等，以便子进程能够正确地从指定位置开始执行。stack是子进程的用户栈顶地址，tf是中断帧或陷阱帧，包含了进程被创建时的CPU状态。

  
#### 5. 将新进程加入全局进程链表
```c
proc->pid = get_pid();
hash_proc(proc);
list_add(&proc_list, &proc->list_link);
nr_process++;
```
- get_pid()：这是一个函数调用，用于获取一个新的进程ID（PID）。
- hash_proc(proc)：将新进程加入到哈希链表中，以便快速查找。
- list_add(&proc_list, &proc->list_link)：将新进程加入到全局进程链表proc_list中。
- nr_process++：全局变量nr_process加1，表示系统中的进程数增加了一个。

#### 6. 将新进程置为可调度状态
```c
wakeup_proc(proc);
```
- wakeup_proc(proc)：这是一个函数调用，用于将新进程从睡眠状态唤醒，并将其置为可调度状态，以便调度器能够选择它进行执行。

#### 7. 设置返回值为子进程的PID
```c
ret = proc->pid;
```
- 将子进程的PID赋值给ret变量，作为函数的返回值。这样，调用者就可以知道新创建的进程的PID了。
 

#### 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。
可以，uCore通过`get_pid`函数和`do_fork`函数中的相关逻辑，可以做到给每个新`fork`的线程（或进程）分配一个唯一的ID。
##### PID分配流程
 
1. **调用`get_pid`函数**：
   - 在`do_fork`函数中，新创建的进程通过调用`get_pid`函数来获取一个PID。
 
2. **`get_pid`函数逻辑**：
   - `get_pid`函数维护了两个静态变量：`last_pid`和`next_safe`。
   - `last_pid`用于跟踪上一个分配的PID，每次调用`get_pid`时都会递增。
   - 如果`last_pid`达到`MAX_PID`，则会回绕到1。
   - `next_safe`用于优化搜索，记录下一个可能的安全PID值。
 
3. **唯一性检查**：
   - `get_pid`函数会遍历进程列表`proc_list`，检查`last_pid`是否与现有进程的PID冲突。
   - 如果冲突，`last_pid`会继续递增，并重新检查，直到找到一个未使用的PID。
   - 通过这种机制，确保分配的PID是唯一的。
 
4. **PID分配与进程创建**：
   - 一旦找到可用的PID，该PID会被赋值给新进程的`proc->pid`。
   - 新进程随后被添加到进程列表`proc_list`中，成为系统的一部分。
 
##### 唯一性保证
 
- **静态变量维护**：`last_pid`和`next_safe`作为静态变量，在`get_pid`函数多次调用之间保持状态，确保PID分配的连续性和唯一性。
- **冲突检测**：通过遍历进程列表并检查PID冲突，`get_pid`函数能够确保分配的PID不与现有进程的PID重复。
- **资源回收**：如果在进程创建过程中发生错误，`do_fork`函数会跳转到相应的错误处理代码块，释放已分配的资源（包括进程结构和内核栈）。由于`last_pid`已经递增，因此失败的PID不会被重新使用。
 


  

### 练习3：编写proc_run 函数（需要编码）
proc_run用于将指定的进程切换到CPU上运行。它的大致执行步骤包括：
- 检查要切换的进程是否与当前正在运行的进程相同，如果相同则不需要切换。
- 禁用中断。你可以使用/kern/sync/sync.h中定义好的宏local_intr_save(x)和local_intr_restore(x)来实现关、开中断。
- 切换当前进程为要运行的进程。
- 切换页表，以便使用新进程的地址空间。/libs/riscv.h中提供了lcr3(unsigned int cr3)函数，可实现修改CR3寄存器值的功能。
- 实现上下文切换。/kern/process中已经预先编写好了switch.S，其中定义了switch_to()函数。可实现两个进程的context切换。
- 允许中断。

请回答如下问题：
- 在本实验的执行过程中，创建且运行了几个内核线程？

完成代码编写后，编译并运行代码：`make qemu`
如果可以得到如 附录A所示的显示内容（仅供参考，不是标准答案输出），则基本正确。

### 扩展练习 Challenge：
- 说明语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`是如何实现开关中断的？

#### 1. `sstatus` 寄存器

在 RISC-V 架构中，`sstatus` 寄存器（Supervisor Status Register）用于保存和控制当前处理器的状态。其中，`SSTATUS_SIE`（Supervisor Interrupt Enable）位用于控制是否允许 **Supervisor 模式** 下的中断。

- **`sstatus` 寄存器**：一个控制和状态寄存器，包含了多个标志位，用于管理处理器的运行状态。
- **`SSTATUS_SIE` 位**：位于 `sstatus` 寄存器中的一个标志位，用于控制是否允许 **Supervisor 模式** 的中断。当该位被设置（1）时，允许中断；当被清除（0）时，禁止中断。

#### 2. 关键函数与宏的作用

##### a. `intr_enable` 和 `intr_disable`

```c
/* intr_enable - enable irq interrupt */
void intr_enable(void) { set_csr(sstatus, SSTATUS_SIE); }

/* intr_disable - disable irq interrupt */
void intr_disable(void) { clear_csr(sstatus, SSTATUS_SIE); }
```

- **`intr_enable`**：通过设置 `sstatus` 寄存器的 `SSTATUS_SIE` 位，开启中断。
- **`intr_disable`**：通过清除 `sstatus` 寄存器的 `SSTATUS_SIE` 位，关闭中断。

**`set_csr` 和 `clear_csr`** 是用于设置和清除 CSR（Control and Status Registers）寄存器中特定位的宏或内联函数。

##### b. `__intr_save` 和 `__intr_restore`

```c
static inline bool __intr_save(void) {
    if (read_csr(sstatus) & SSTATUS_SIE) {
        intr_disable();
        return 1;
    }
    return 0;
}

static inline void __intr_restore(bool flag) {
    if (flag) {
        intr_enable();
    }
}
```

- **`__intr_save`**：
  - **作用**：读取当前 `sstatus` 寄存器的 `SSTATUS_SIE` 位状态，判断中断是否已开启。
  - **逻辑**：
    1. 使用 `read_csr(sstatus)` 读取 `sstatus` 寄存器的当前值。
    2. 检查 `SSTATUS_SIE` 位是否被设置。
    3. 如果中断已开启（`SSTATUS_SIE` 位为1），则调用 `intr_disable()` 关闭中断，并返回 `1`（表示之前中断是开启的）。
    4. 如果中断未开启，直接返回 `0`（表示之前中断是关闭的）。
  
- **`__intr_restore`**：
  - **作用**：根据传入的标志位 `flag`，决定是否重新开启中断。
  - **逻辑**：
    1. 如果 `flag` 为 `1`，调用 `intr_enable()` 重新开启中断。
    2. 如果 `flag` 为 `0`，则保持中断关闭状态。

##### c. `local_intr_save` 和 `local_intr_restore`

```c
#define local_intr_save(x)      do { x = __intr_save(); } while (0)
#define local_intr_restore(x)   __intr_restore(x);
```

- **`local_intr_save(x)`**：
  - **作用**：调用 `__intr_save()`，将中断的当前状态保存到变量 `x` 中。
  - **实现**：通过 `do { ... } while (0)` 结构，确保宏在使用时行为与函数类似，不会引起语法问题。

- **`local_intr_restore(x)`**：
  - **作用**：调用 `__intr_restore(x)`，根据变量 `x` 的值恢复中断状态。
  - **实现**：直接调用 `__intr_restore`，无需 `do-while` 结构。

#### 3. 具体实现过程详解

让我们详细跟踪 `local_intr_save(intr_flag);` 和 `local_intr_restore(intr_flag);` 的执行过程，以理解它们如何实现中断的开启与关闭。

##### a. `local_intr_save(intr_flag);`

1. **调用宏**：
   ```c
   local_intr_save(intr_flag);
   ```
   
2. **展开宏**：
   ```c
   do {
       intr_flag = __intr_save();
   } while (0);
   ```
   
3. **执行 `__intr_save()`**：
   - **读取 `sstatus` 寄存器**：
     ```c
     if (read_csr(sstatus) & SSTATUS_SIE) {
     ```
     - `read_csr(sstatus)` 获取 `sstatus` 寄存器的当前值。
     - `& SSTATUS_SIE` 检查中断是否开启。
   
   - **条件判断**：
     - **如果中断已开启**：
       ```c
       intr_disable();
       return 1;
       ```
       - 调用 `intr_disable()` 关闭中断。
       - 返回 `1`，表示中断之前是开启的。
   
     - **如果中断未开启**：
       ```c
       return 0;
       ```
       - 不做任何操作，返回 `0`，表示中断之前是关闭的。
   
4. **保存状态**：
   - 将 `__intr_save()` 返回的值（`1` 或 `0`）赋值给 `intr_flag`，用于后续恢复中断状态。

##### b. `local_intr_restore(intr_flag);`

1. **调用宏**：
   ```c
   local_intr_restore(intr_flag);
   ```
   
2. **展开宏**：
   ```c
   __intr_restore(intr_flag);
   ```
   
3. **执行 `__intr_restore(intr_flag)`**：
   - **条件判断**：
     ```c
     if (flag) {
         intr_enable();
     }
     ```
     - 如果 `flag` 为 `1`，调用 `intr_enable()` 重新开启中断。
     - 如果 `flag` 为 `0`，保持中断关闭状态。

#### 4. 总结与作用

通过上述过程，我们可以看出 `local_intr_save(intr_flag);` 和 `local_intr_restore(intr_flag);` 的作用如下：

1. **保存中断状态**：
   - `local_intr_save(intr_flag);` 通过 `__intr_save()` 检查当前中断是否开启。
   - 如果中断开启，则关闭中断并记录原先的状态（开启）。
   - 如果中断关闭，则记录原先的状态（关闭）。

2. **临界区保护**：
   - 在调用 `local_intr_save(intr_flag);` 之后，中断被关闭，可以安全地进入临界区，执行需要原子性保障的操作，避免被中断打断。

3. **恢复中断状态**：
   - `local_intr_restore(intr_flag);` 根据之前保存的状态，决定是否重新开启中断。
   - 如果原先中断是开启的，则重新开启中断；如果原先中断是关闭的，则保持中断关闭状态。

