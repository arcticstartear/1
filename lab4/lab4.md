# lab4
## 练习一：
### alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。
### 【提示】在alloc_proc函数的实现中，需要初始化的proc_struct结构中的成员变量至少包括：state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name。
### 请在实验报告中简要说明你的设计实现过程。
答：根据题目中提示和相关英文注解，完成对proc_struct结构体成员变量的初始化如下
![](https://pic.imgdb.cn/item/655b4bdfc458853aeffc4a60.png)
### 请回答如下问题：
### 请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）

答：其含义和作用如下：

·struct context context

上下文结构体，通常用于保存进程或线程在执行过程中的上下文信息，包括寄存器值、指令指针等。在操作系统中，当进行进程切换时，需要保存当前进程的上下文信息，并将其恢复到另一个进程的上下文信息，以实现进程间的切换。因此，上下文结构体对于实现进程调度和切换非常重要。

·struct trapframe *tf

指向当前中断的陷阱帧，陷阱帧是在处理中断或异常时保存的CPU状态的数据结构。它包含了被中断程序的寄存器状态、中断原因以及其他与中断处理相关的信息。指向当前中断的陷阱帧的指针通常在内核中使用，以便内核能够获取并处理当前中断的相关信息。在处理完中断后，可以使用陷阱帧中保存的信息来恢复被中断程序的执行状态。
## 练习二： 
### 创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。因此，我们实际需要”fork”的东西就是stack和trapframe。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：
### ·调用alloc_proc，首先获得一块用户信息块。
### ·为进程分配一个内核栈。
### ·复制原进程的内存管理信息到新进程（但内核线程不必做此事）
### ·复制原进程上下文到新进程
### ·将新进程添加到进程列表
### ·唤醒新进程
### ·返回新进程号
### 请在实验报告中简要说明你的设计实现过程。
答：代码流程如下，根据题中提示的步骤完成，并配有详细注释
![](https://pic.imgdb.cn/item/655b4be0c458853aeffc4bcb.png)
### 请回答如下问题：
### 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。
ucore在进行进程（线程）创建时，会为每个新fork的线程分配一个唯一的ID，即进程ID（PID）。这样可以确保每个进程都有一个唯一的标识符：

创建进程的函数do_fork会在创建新进程时，分配一个新的进程结构体，并通过get_pid()函数为每个新fork的线程分配了一个唯一的pid，保证了每个进程在系统中具有唯一的标识符。
## 练习三：
### proc_run用于将指定的进程切换到CPU上运行。它的大致执行步骤包括：
### ·检查要切换的进程是否与当前正在运行的进程相同，如果相同则不需要切换。
### ·禁用中断。你可以使用/kern/sync/sync.h中定义好的宏local_intr_save(x)和local_intr_restore(x)来实现关、开中断。
### ·切换当前进程为要运行的进程。
### ·切换页表，以便使用新进程的地址空间。/libs/riscv.h中提供了lcr3(unsigned int cr3)函数，可实现修改CR3寄存器值的功能。
### ·实现上下文切换。/kern/process中已经预先编写好了switch.S，其中定义了switch_to()函数。可实现两个进程的context切换。
### ·允许中断。
答：根据题中提示完成代码，并标注注释信息
![](https://pic.imgdb.cn/item/655b4be0c458853aeffc4d32.png)
### 请回答如下问题：
### 在本实验的执行过程中，创建且运行了几个内核线程？
创建并运行了两个内核线程，分别是idleproc和initproc：

idleproc是系统中的空闲进程，当系统没有其他可运行的进程时，它会被调度执行。idleproc的主要作用是占用CPU资源，以防止系统处于空转或浪费资源的状态。它通常在系统启动时创建，并在整个系统运行期间一直存在；

initproc是系统中的初始化进程，它是所有其他进程的祖先进程。在系统启动完成后，内核会通过fork()函数创建一个新的进程，并将其父进程设置为initproc。initproc负责启动和管理系统中的各种服务和进程，它是系统的第一个用户级进程，通常具有较高的权限，在本实验中initproc只需要打印显示证明其功能实现无误即可。
## Challenge：
### 说明语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);是如何实现开关中断的？
首先是local_intr_save：
![](https://pic.imgdb.cn/item/655b4be1c458853aeffc4ec4.png)
当调用local_intr_save(intr_flag)时，会展开为：
```
do {
    intr_flag = __intr_save();
} while (0);
```
这里调用了__intr_save()函数来保存当前的中断状态，并将其值赋给intr_flag变量。

接着是local_intr_restore：
![](https://pic.imgdb.cn/item/655b4be1c458853aeffc4f90.png)
当调用local_intr_restore(intr_flag)时，会展开为：
```
__intr_restore(intr_flag);
```
这里直接调用__intr_restore函数，并将之前保存的中断状态intr_flag作为参数传递给该函数。

__intr_save函数和__intr_restore函数则通过操作RISC-V架构中的特权寄存器来实现对中断的保存和恢复。

下面是__intr_save函数和__intr_restore函数具体内容

**__intr_save函数：**

·首先通过read_csr(sstatus)读取当前的 sstatus 寄存器的值；

·然后使用按位与操作符&和SSTATUS_SIE（表示 Supervisor Interrupt Enable）来检查当前是否允许中断（即 sstatus 寄存器中 SIE 位是否被置位）；

·如果允许中断，则调用intr_disable()函数来禁止中断，并返回true；否则返回false。
![](https://pic.imgdb.cn/item/655b4e67c458853aef0674d3.png)
**__intr_restore函数：**

·接受一个布尔类型的参数flag，表示之前保存的中断状态；

·如果flag为true，即之前允许中断，则调用intr_enable()函数来重新允许中断。
![](https://pic.imgdb.cn/item/655b4e67c458853aef067551.png)
命令make grade结果：
![](https://pic.imgdb.cn/item/655b4e67c458853aef067579.png)