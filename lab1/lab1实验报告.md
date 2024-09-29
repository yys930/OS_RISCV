# 2213781
# Lab0.5: 比麻雀更小的麻雀（最小可执行内核）
## 练习1：为了熟悉使用qemu和gdb进行调试工作,使用gdb调试QEMU模拟的RISC-V计算机加电开始运行到执行应用程序的第一条指令（即跳转到0x80200000）这个阶段的执行过程，说明RISC-V硬件加电后的几条指令在哪里？完成了哪些功能？要求在报告中简要写出练习过程和回答。

1. 调试：分别打开两个终端，一个输入make debug，一个输入make gdb，即可使用gdb远程调试QEMU模拟的RISC-V计算机。
2. 进入gdb后，输入x/10i $pc可以查看显示即将执行的10条汇编指令，如下所示，可以看到QEMU刚启动时的地址为0x1000，这是QEMU模拟的RISCV处理器的复位地址，即CPU在上电的时候，或者按下复位键的时候，PC被赋的初始值。紧接着，QEMU会执行后面的汇编指令，这些指令是引导代码指令，负责完成一些初始化工作，如获取核心ID、初始化一些寄存器、进行硬件自检等，同时，作为 bootloader 的 OpenSBI.bin 被加载到物理内存以物理地址 0x80000000 开头的区域上，同时内核镜像 os.bin 被加载到以物理地址 0x80200000 开头的区域上，然后，程序计数器即PC会被设定为地址0x80000000，即引导加载程序bootloader的起始地址。
````
   0x1000:      auipc   t0,0x0
   0x1004:      addi    a1,t0,32
   0x1008:      csrr    a0,mhartid
   0x100c:      ld      t0,24(t0)
   0x1010:      jr      t0
   0x1014:      unimp
   0x1016:      unimp
   0x1018:      unimp
   0x101a:      0x8000
   0x101c:      unimp
````
3. PC跳转到0x80000000的地址后，将会加载引导加载程序即bootloader：OpenSBI.bin到内存中，该程序将完成硬件初始化、自检和诊断、设置执行环境等一系列初始化任务，同时，该程序会将PC的值会被设定为 0x80200000，这是操作系统内核的起始地址。
4. 在 0x80200000地址，操作系统内核镜像os.bin会加载到内存中，操作系统内核将进行更进一步的初始化工作，如准备内存、初始化数据结构、初始化硬件、加载驱动程序等，等工作完成后，PC将被设置为应用程序的第一行代码地址，正式启动和执行应用程序。

---



# Lab1:断,都可以断
## 练习1：理解内核启动中的程序入口操作
1. `la sp, bootstacktop`：这条指令将 bootstacktop 的地址加载到堆栈指针（sp）寄存器中。目的如下：
> * 初始化堆栈：将堆栈指针指向内核的启动堆栈顶部，这为后续的函数调用和局部变量的存储做好准备。
> * 设置堆栈空间：在操作系统内核启动过程中，使用堆栈是执行函数调用和处理中断的重要组成部分，这条指令可以确保内核在运行时有足够的空间来管理局部数据和控制流。
2. `tail kern_init：tail`是 RISC-V 汇编中用于优化的一个指令，表示将控制权转移到 kern_init 函数。这里的 tail 意味着该指令将直接跳转到 kern_init，并且不需要返回，因此可以优化栈帧。目的：
> * 执行内核初始化：kern_init 是内核启动过程中的关键初始化函数，负责设置内核的数据结构、初始化设备驱动、配置内存管理、调度等。
> * 流畅过渡：通过 tail 跳转，避免创建新的栈帧，使得内核的启动过程更加高效，直接将控制权传递给初始化过程。

总结：
* `la sp, bootstacktop` 初始化堆栈指针，为内核操作提供必要的堆栈空间。
* `tail kern_init` 将控制权转移到内核初始化函数，开始系统的初始化过程，并通过优化使得控制流更加高效。

## 练习2：完善中断处理 （需要编程）
完善代码如下，首先调用在clock.c文件中定义的clock_set_next_event()函数，设置下一次时钟中断，然后定义一个静态变量ticks负责计数，通过与TICK_NUM进行取模操作判断是否达到100次，如果符合条件，就通过调用print_ticks()输出提示信息，同时打印次数num加1，当num等于10时，输出关机提示，并调用sbi.h中的关机函数sbi_shutdown()关机。
````
case IRQ_S_TIMER:
            // "All bits besides SSIP and USIP in the sip register are
            // read-only." -- privileged spec1.9.1, 4.1.4, p59
            // In fact, Call sbi_set_timer will clear STIP, or you can clear it
            // directly.
            // cprintf("Supervisor timer interrupt\n");
             /* LAB1 EXERCISE2   YOUR CODE :2213781  */
            /*(1)设置下次时钟中断- clock_set_next_event()
             *(2)计数器（ticks）加一
             *(3)当计数器加到100的时候，我们会输出一个`100ticks`表示我们触发了100次时钟中断，同时打印次数（num）加一
            * (4)判断打印次数，当打印次数为10时，调用<sbi.h>中的关机函数关机
            */
            // (1) 设置下次时钟中断
            clock_set_next_event(); 

            // (2) 计数器加一
            static int ticks = 0;
            ticks++;

            // (3) 当计数器达到100次时输出提示信息
            if (ticks % TICK_NUM == 0) {
                print_ticks();
                num++;
            }

            // (4) 当打印次数达到10次时关机
            if (num == 10) {
                cprintf("Shutting down...\n");
                sbi_shutdown();
            }
    
            break;
````
输出结果如下：
````
Special kernel symbols:
  entry  0x000000008020000a (virtual)
  etext  0x0000000080200a08 (virtual)
  edata  0x0000000080204010 (virtual)
  end    0x0000000080204030 (virtual)
Kernel executable memory footprint: 17KB
++ setup timer interrupts
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
Shutting down...
````

## 扩展练习 Challenge1：描述与理解中断流程
### 处理中断异常的流程：
1. 异常或中断的产生：当硬件或软件条件触发异常（如非法指令、页错误、外部中断等），处理器会自动保存当前上下文信息，并跳转到特定的异常处理向量（如 __alltraps），进入异常处理程序。
2. 保存当前状态：在异常处理程序中（即__alltraps），首先会调用 SAVE_ALL 宏来保存当前处理器的状态。SAVE_ALL 宏中执行 `csrw sscratch, sp` 将当前的堆栈指针 sp 保存到 sscratch 寄存器，以便在异常处理过程中使用。然后将 sp 向下移动以为寄存器保存分配空间，接着依次保存所有通用寄存器（x0 到 x31）及特定状态寄存器（如 sstatus, sepc, sbadaddr, scause）到栈中。
3. 处理异常：通过 `move a0, sp `将堆栈指针传递给异常处理函数 trap，然后通过`jal trap`调用trap函数处理异常。
4. 恢复寄存器状态：异常处理完成后，执行 RESTORE_ALL 宏，恢复之前保存的寄存器状态，确保处理完异常后能正确返回到之前的状态。
5. 返回：最后执行 `sret` 指令返回到之前的执行上下文。

### `mov a0, sp` 的目的是什么？
`mov a0, sp` 的目的是将当前的堆栈指针值传递给异常处理函数 trap。在 trap 函数中，a0 寄存器通常用于传递参数，因此这里的操作确保了异常处理代码能够访问保存的上下文信息。

### SAVE_ALL 中寄存器保存的位置是如何确定的？
SAVE_ALL 中寄存器的保存位置是通过堆栈指针 sp 来动态确定的。在保存寄存器之前，首先将 sp 向下调整（addi sp, sp, -36 * REGBYTES），然后在此基础上以相对偏移量存储各个寄存器的值。每个寄存器的存储位置由其在 SAVE_ALL 宏中的顺序和所用的偏移量决定。

### 对于任何中断，__alltraps 中都需要保存所有寄存器吗？说明理由。
是的，__alltraps 中需要保存所有寄存器。这是因为在处理异常或中断时，处理器的状态（包括寄存器的内容）必须被完全保存，以便在处理完成后能恢复到之前的状态。如果不保存所有寄存器，一些寄存器的内容可能会在异常处理过程中被覆盖，从而导致程序状态的不一致和不可预料的行为。因此，保存所有寄存器是确保系统稳定性和正确性的重要步骤。

## 扩展练习 Challenge2：理解上下文切换机制
### `csrw sscratch, sp；csrrw s0, sscratch, x0`
1. `csrw sscratch, sp`：
> * 作用：将当前的堆栈指针（sp）的值写入到控制状态寄存器 sscratch 中。
> * 目的：这个操作的目的是保存当前上下文的堆栈指针，以便在异常处理过程中能够恢复堆栈状态。sscratch 寄存器的值可以在发生递归异常时帮助识别是否是从内核态进入的异常。
2. `csrrw s0, sscratch, x0`：
> * 作用：将 sscratch 寄存器的值读入到 s0 中，并将 x0 的值（0）写入到 sscratch 中。
> * 目的：将 sscratch 的值存储到 s0 中，并将 sscratch 清零，以便后续的异常处理能够判断是否发生了递归异常。清零后，如果再次发生异常，系统可以知道它是从内核中再次触发的。

### 关于保存和恢复 CSR
在 SAVE_ALL 中，stval, scause 等 CSR 寄存器被保存，而在 RESTORE_ALL 中并没有还原它们的操作。原因如下：
1. 保存的意义：保存 stval 和 scause 是为了在处理完异常后能够分析异常原因。这些信息可以在异常处理代码中使用，例如用于调试或日志记录。
2. 不还原的理由：在恢复的过程中，不需要将这些 CSR 寄存器还原为之前的值，因为在异常处理完成后，处理器状态可能会发生变化，并且这些寄存器的值通常与上一个上下文无关。处理器会根据当前的异常状态来设置它们。此外，许多异常处理完后，可能会切换到新的上下文或线程，因此在返回时，CSR 寄存器的值通常是由新的上下文管理的。

### 总结：
SAVE_ALL 和 RESTORE_ALL 设计的目的是为了确保在处理中断和异常时能正确保存和恢复通用寄存器的状态，同时 stval 和 scause 的保存则是为了后续处理或调试的需要，而不需要在恢复时再还原它们。

## 扩展练习Challenge3：完善异常中断
### 完善的代码如下，首先调用该系统封装的打印函数cprintf打印指令异常类型，然后打印tf->epc寄存器的值，输出异常指令地址，最后更新tf->epc寄存器的值到下一个地址。这里介绍以下trapframe结构体实例tf的组成，tf中包含了4个uintptr_t类型的变量：status（处理器的状态寄存器（SSTATUS））、epc（异常程序计数器（EPC），保存异常时的程序计数器）、badvaddr（错误地址（Bad Virtual Address），用于指示错误发生的地址（在部分异常中使用））和cause（异常原因寄存器（CAUSE），指示触发异常或中断的原因）。
````
 case CAUSE_ILLEGAL_INSTRUCTION:
             // 非法指令异常处理
             /* LAB1 CHALLENGE3   YOUR CODE :2213781  */
            /*(1)输出指令异常类型（ Illegal instruction）
             *(2)输出异常指令地址
             *(3)更新 tf->epc寄存器
            */
            cprintf("Exception type: Illegal instruction\n");

            cprintf("Exception address (EPC): 0x%lx\n", tf->epc);

            tf->epc += 4;
            break;
        case CAUSE_BREAKPOINT:
            //断点异常处理
            /* LAB1 CHALLLENGE3   YOUR CODE :2213781  */
            /*(1)输出指令异常类型（ breakpoint）
             *(2)输出异常指令地址
             *(3)更新 tf->epc寄存器
            */
            cprintf("Exception type: breakpoint\n");

            cprintf("Exception address (EPC): 0x%lx\n", tf->epc);

            tf->epc += 4;
  

            break;
````

### 触发异常的代码：`__asm__ volatile("mret")`可以触发非法指令异常，`__asm__ volatile("ebreak")`可以触发断点异常。将这两句代码分别插入时钟中断的初始化函数clock_init()中，即可在设置时钟中断的同时，设置这两种异常。结果如下：
* 非法指令异常
代码：
````
void clock_init(void) {
    // enable timer interrupt in sie
    set_csr(sie, MIP_STIP);
    // divided by 500 when using Spike(2MHz)
    // divided by 100 when using QEMU(10MHz)
    // timebase = sbi_timebase() / 500;
    clock_set_next_event();

    // initialize time counter 'ticks' to zero
    ticks = 0;

    cprintf("++ setup timer interrupts\n");

    __asm__ volatile("mret"); // 触发非法指令异常
    //__asm__ volatile("ebreak");  // 触发断点异常
}
````
结果：
````
++ setup timer interrupts
sbi_emulate_csr_read: hartid0: invalid csr_num=0x302
Exception type: Illegal instruction
Exception address (EPC): 0x8020015e
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
Shutting down...
````
* 断点异常：
代码：
````
void clock_init(void) {
    // enable timer interrupt in sie
    set_csr(sie, MIP_STIP);
    // divided by 500 when using Spike(2MHz)
    // divided by 100 when using QEMU(10MHz)
    // timebase = sbi_timebase() / 500;
    clock_set_next_event();

    // initialize time counter 'ticks' to zero
    ticks = 0;

    cprintf("++ setup timer interrupts\n");

    //__asm__ volatile("mret"); // 触发非法指令异常
    __asm__ volatile("ebreak");  // 触发断点异常
}
````
````
++ setup timer interrupts
Exception type: breakpoint
Exception address (EPC): 0x8020015e
Exception type: breakpoint
Exception address (EPC): 0x8020015e
Exception type: breakpoint
Exception address (EPC): 0x8020015e
Exception type: breakpoint
Exception address (EPC): 0x8020015e
````
