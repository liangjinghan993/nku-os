# Lab1 实验报告

## 练习1 理解内核启动中的程序入口操作

### 阅读 kern/init/entry.S 内容代码，结合操作系统内核启动流程，说明指令 la sp, bootstacktop 完成了什么操作，目的是什么？tail kern\_init 完成了什么操作，目的是什么？

指令 `la sp, bootstacktop` 的作用是将栈指针（Stack Pointer，sp）设置为 bootstacktop 的地址。这里的 bootstacktop 是一个标签，代表内核栈的顶部位置。

在操作系统内核启动流程中，设置栈指针的目的是为了正确地初始化内核栈。内核栈是用来保存函数调用过程中的局部变量、参数和返回地址等信息的一块内存区域。通过将栈指针设置为内核栈的顶部，可以确保函数调用时的栈操作在正确的内存位置进行。

指令 `tail kern_init` 的作用是跳转到标签 kern\_init 处执行代码。这里的 kern\_init 是内核初始化函数的入口点。

在操作系统内核启动流程中，跳转到内核初始化函数的目的是开始执行操作系统的初始化过程。内核初始化函数负责完成各种初始化任务，例如初始化设备驱动程序、建立内存管理机制、创建进程或线程等。

因此，指令 `la sp, bootstacktop` 设置了栈指针，而 `tail kern_init` 跳转到内核初始化函数，这两个操作共同完成了操作系统内核的初始化准备工作。

## 练习2 完善中断处理（需要编程）

### 请编程完善 trap.c 中的中断处理函数 trap，在对时钟中断进行处理的部分填写 kern/trap/trap.c 函数中处理时钟中断的部分，使操作系统每遇到 100 次时钟中断后，调用 print\_ticks 子程序，向屏幕上打印一行文字”100ticks”，在打印完 10 行后调用 sbi.h 中的 shut\_down() 函数关机。

### 要求完成问题 1 提出的相关函数实现，提交改进后的源代码包（可以编译执行），并在实验报告中简要说明实现过程和定时器中断中断处理的流程。实现要求的部分代码后，运行整个系统，大约每 1 秒会输出一次”100 ticks”，输出 10 行。

#### 代码实现：

```c
case IRQ_S_TIMER:
    clock_set_next_event();  // 设置下次时钟中断

    if (++ticks % TICK_NUM == 0) {
        print_ticks();  // 打印 "100 ticks"

        if (++num % 10 == 0) {
            cprintf("Supervisor timer interrupt\n");
            sbi_shutdown();  // 关机
        }
    }
    break;

```

#### 实现过程：

1.  `clock_set_next_event()`：设置下次时钟中断的触发时间。
2.  `++ticks % TICK_NUM == 0`：`ticks` 是一个计数器，每次时钟中断发生时增加 1。这个条件判断语句检查 `ticks` 的值是否是 `TICK_NUM` 的倍数，也就是是否达到了 100 次时钟中断。ticks的声明在kern\driver\clock.h文件中的extern volatile size\_t ticks;
3.  `print_ticks()`：如果 `ticks` 达到了 `TICK_NUM` 的倍数，即 100 次时钟中断，调用 `print_ticks()` 函数，打印 "100 ticks"
4.  `++num % 10 == 0`：`num` 是一个计数器，记录打印次数。每次打印 "100 ticks" 时，`num` 增加 1。这个条件判断语句检查 `num` 的值是否是 10 的倍数，也就是是否达到了 10 次打印。num的声明在kern\trap\trap.c文件中的volatile size\_t num=0;
5.  `cprintf("Supervisor timer interrupt\n")`：如果 `num` 达到了 10 的倍数，即 10 次打印，打印 "Supervisor timer interrupt"
6.  `sbi_shutdown()`：如果 `num` 达到了 10 的倍数，调用\<sbi.h>中的关机函数void sbi\_shutdown(void)关机

#### 定时器中断中断处理流程

1.  当定时器中断发生时，处理器会保存当前的执行状态（包括寄存器值和程序计数器等）
2.  处理器会跳转到预先设置好的中断处理函数，即上述代码片段中的 `case IRQ_S_TIMER:`
3.  中断处理函数根据具体的需求进行相应的处理。对于定时器中断，常见的处理操作包括更新计时器、处理定时任务、调度任务等。`clock_set_next_event()` 函数会设置下次时钟中断的触发时间；`++ticks` 会将 `ticks` 的值增加 1，并使用 `%` 运算符判断是否达到 `TICK_NUM` 的倍数。如果达到，则执行下面的代码块；在达到 `TICK_NUM` 的倍数时，`print_ticks()` 函数会被调用，打印 "100 ticks"；`++num` 会将 `num` 的值增加 1，并使用 `%` 运算符判断是否达到 10 的倍数。如果达到，则执行下面的代码块。
4.  处理完中断后，中断处理函数会恢复之前保存的执行状态，包括恢复寄存器的值和程序计数器等
5.  处理器会从中断帧中恢复之前的执行状态，继续执行被中断的程序或任务。由于定时器中断是内核级别的，因此不需要恢复进程的上下文信息，直接返回即可。

#### 实现截图：

![74066fe463c51adcc38e911442e4b57.jpg](https://gitee.com/liang-jinghan888/nku-operating-system-2023/raw/master/%E5%9B%BE%E7%89%87%E6%96%87%E4%BB%B6%E5%A4%B9/1-1.png)

## Challenge1：描述与理解中断流程

### 描述 ucore 中处理中断异常的流程（从异常的产生开始），其中 mov a0，sp 的目的是什么？SAVE\_ALL中寄寄存器保存在栈中的位置是什么确定的？对于任何中断，\_\_alltraps 中都需要保存所有寄存器吗？请说明理由

#### ucore 中处理中断异常的流程

1.  中断和异常时，CPU保存当前进程的上下文（寄存器值等），以便在处理完中断或异常后能够恢复到之前的执行状态。
2.  CPU会跳转到预先设置好的中断处理函数（\_\_alltraps函数）
3.  \_\_alltraps函数将栈指针sp值保存到寄存器a0中，调用trap（）处理中断或异常
4.  trap（）函数根据中断异常类型进行处理。系统调用就调用相应的系统调用处理函数，中断就调用中断处理函数，异常就打印错误信息调用panic（）函数
5.  处理函数执行完成后，会根据之前保存的上下文信息恢复执行状态，返回到中断或异常发生的地方，继续执行被中断的代码或者进行异常处理。

### `mov a0，sp` 的目的

`mov a0，sp` 将栈指针sp的值保存到寄存器a0中，以便在trap（）函数中能够正确恢复到被打断进程的上下文。 在栈中保存了被打断进程的上下文信息，需要将栈指针保存在寄存器中，以便在恢复上下文信息时能够正确访问栈中数据。

### `SAVE_ALL`中寄寄存器保存在栈中的位置是什么确定的？

`SAVE_ALL` 是一个通常用于中断处理的例程，用于保存所有寄存器的当前值到栈中，以便在中断处理完成后可以恢复它们的值。在`SAVE_ALL`中，寄存器保存在栈中的位置是通过栈指针 sp 和偏移量（即寄存器编号乘以 4）计算得出的

### `__alltraps` 中都需要保存所有寄存器吗？

 对于任何中断，`__alltraps` 函数都需要保存所有寄存器。这是因为在处理中断时，需要保证被打断进程的上下文信息不受影响。如果不保存所有寄存器，可能会导致被打断进程的部分上下文信息丢失，从而导致无法正确地恢复进程执行。因此，在处理中断时，需要保存所有寄存器。

## Challenge2 理解上下文切换机制

### 在 trapentry.S 中汇编代码 csrw sscratch, sp；csrrw s0, sscratch, x0 实现了什么操作，目的是什么？save all里面保存了 stval scause 这些 csr，而在 restore all 里面却不还原它们？那这样 store 的意义何在呢？

`csrw sscratch, sp`：这行汇编代码将当前线程的栈指针 `sp` 的值写入 `sscratch` 控制寄存器。这通常用于保存当前线程的栈指针，以便在后续的代码中可以恢复它。这在中断或异常处理时很有用，因为它允许线程在中断/异常处理完成后恢复其之前的上下文。在发生中断时，CPU 会自动将当前的上下文信息保存在`sscratch`中

`csrrw s0, sscratch, x0`：将状态寄存器`sscratch` 的值读出到寄存器`s0`中，并将`sscratch`的值设置为 0。其目的是告诉中断处理程序异常来自内核，因为在内核态下，`sscratch`的值应该为 0。这个操作实际上是一种线程上下文切换的一部分。它用于保存当前线程的 `sscratch` 寄存器的值，然后将其替换为新线程的 `sscratch` 寄存器的值，以切换到新线程的上下文。

在操作系统内核中，通常会将某些异常信息保存在控制寄存器中，以便在异常处理程序中进行分析和记录。这些寄存器包括：

*   `stval`：用于保存导致异常的指令的附加信息，例如访问无效地址时的地址值。
*   `scause`：用于保存导致异常的原因，如中断、陷阱或其他异常类型。

这些寄存器在异常处理程序中非常有用，因为它们提供了有关发生异常的详细信息。然而，在上下文切换期间，操作系统可能会决定不还原这些寄存器的值，因为它们通常是特定于异常处理的，而不是线程上下文的一部分。当操作系统切换到不同的线程时，它通常会希望新线程的异常处理程序有自己的 `stval` 和 `scause` 值，以便正确地处理异常

在这种情况下，"store" 操作的意义在于保存当前线程的异常相关状态，以便在后续的异常处理中使用。而 "restore" 操作则用于加载新线程的异常相关状态。这种设计使得异常处理能够在不同线程之间正确工作，因为每个线程都有自己的异常处理上下文。

总之，"save" 和 "restore" 操作的目的是为了在线程上下文切换时正确处理异常，并确保每个线程都有自己的异常处理状态。这有助于确保操作系统的稳定性和可靠性。

## Challenge3 完善异常中断

### 编程完善在触发一条非法指令异常 mret 和，在 kern/trap/trap.c 的异常处理函数中捕获，并对其进行处理，简单输出异常类型和异常指令触发地址，即“Illegal instruction caught at 0x(地址)”，“ebreak caught at 0x（地址）”与“Exception type\:Illegal instruction”，“Exception type: breakpoint”。

#### 代码实现：

```c
kern\init\init.C
int kern_init(void) {
    extern char edata[], end[];
    memset(edata, 0, end - edata);

    cons_init();  // init the console

    const char *message = "(THU.CST) os is loading ...\n";
    cprintf("%s\n\n", message);

    print_kerninfo();

    // grade_backtrace();

    idt_init();  // init interrupt descriptor table

    // rdtime in mbare mode crashes
    clock_init();  // init clock interrupt

    intr_enable();  // enable irq interrupt

    asm volatile("ebreak"::);
     
    while (1)
        ;
}
```

```c
kern\trap\trap.C
void exception_handler(struct trapframe *tf) {
    switch (tf->cause) {
        case CAUSE_MISALIGNED_FETCH:
            break;
        case CAUSE_FAULT_FETCH:
            break;
        case CAUSE_ILLEGAL_INSTRUCTION:
            cprintf("Illegal instruction caught at 0x%x\n",tf->epc);
           cprintf("Exception type: Illegal instruction\n");
           tf->epc+=2;
            break;
        case CAUSE_BREAKPOINT:
           cprintf("ebreak caught at 0x%x\n",tf->epc);
           cprintf("Exception type: breakpoint\n");
           tf->epc+=2;
            break;
        case CAUSE_MISALIGNED_LOAD:
            break;
        case CAUSE_FAULT_LOAD:
            break;
        case CAUSE_MISALIGNED_STORE:
            break;
        case CAUSE_FAULT_STORE:
            break;
        case CAUSE_USER_ECALL:
            break;
        case CAUSE_SUPERVISOR_ECALL:
            break;
        case CAUSE_HYPERVISOR_ECALL:
            break;
        case CAUSE_MACHINE_ECALL:
            break;
        default:
            print_trapframe(tf);
            break;
    }
}
```

#### 实现截图

![image.png](https://note.youdao.com/yws/res/239/WEBRESOURCE85eb70a9ba1647f6a8ab670d0d68932e)

## 本实验重要知识点

1.  中断处理流程
2.  RISC-V中断处理机制

## OS重要知识点

1.  进程管理：操作系统负责管理计算机上运行的各个进程。包括创建、调度、暂停、恢复和终止。操作系统需要确保进程之间的资源共享和互斥，以避免冲突和竞争条件。
2.  内存管理：操作系统管理计算机的物理和虚拟内存。它需要分配和回收内存，以便进程能够访问所需的内存空间，同时避免内存泄漏和碎片化。
3.  文件系统管理：操作系统负责管理文件和目录，包括文件的创建、读取、写入、删除和权限控制。它还需要确保数据的持久性和一致性。
4.  设备管理：操作系统与硬件设备进行通信，以管理输入输出（I/O）设备。这包括驱动程序的加载、设备的初始化、I/O队列的管理等。

