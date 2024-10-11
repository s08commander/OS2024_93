##  <center> 实验报告</center>
### <center>小组成员</center>
张涛丹 2212009        
申展2211321        
克梦宇2211765
## 一.Lab0.5 实验过程
### （1）启动qemu，连接gdb
首先进入代码所在的目录
<pre><code class="language-bash">cd ~/OS/riscv64-ucore-labcodes/lab0
</code></pre>
用qemu编译代码，并等待gdb连接
<pre><code class="language-bash">make debug
</code></pre>
打开一个新的终端窗口，将gdb连接到qemu
<pre><code class="language-bash">make gdb
</code></pre>
### （2）用gdb调试可执行文件
首先查看通电后执行的10条汇编指令，在gdb输入指导书提供的指令
<pre><code class="language-bash">x/10i $pc
</code></pre>
<pre><code class="language-asm">0x1000:      auipc   t0,0x0
0x1004:      addi    a1,t0,32
0x1008:      csrr    a0,mhartid
0x100c:      ld      t0,24(t0)
0x1010:      jr      t0
0x1014:      unimp
0x1016:      unimp
0x1018:      unimp
0x101a:      0x8000
0x101c:      unimp
</code></pre>
和指导书中的内容符合，能看出PC的初始值为0x1000。同时发现在0x1010处是一条跳转指令。用指令si逐条运行到这条汇编指令，再用指令info r t0查看寄存器的值，结果为0x80000000。这是OpenSBI的入口地址，接下来它会将位于0x80200000的操作系统加载到内存中。

## 二.Lab1 
### （1）练习1
指令 la sp, bootstacktop将 bootstacktop 的地址加载到栈指针寄存器 sp 中。栈指针将指向内核栈的顶部（bootstacktop）。目的是初始化栈指针 `sp`，将其设置为内核的栈空间，这样后续的函数调用和局部变量的使用都可以正常进行。
指令tail kern_init 从 kern_entry 跳转到 kern_init，开始执行kern_init，并且不返回kern_entry。这样做的目的是直接进入内核的初始化阶段。
### （2）练习2
完善中断处理函数：
```
case IRQ_S_TIMER:
            clock_set_next_event();
            if (++ticks % TICK_NUM == 0)
            {
                print_ticks();
            }
            break;
```
调用clock_set_next_event()函数设置下一次时钟中断事件，判断++ticks是否为100，即是否遇到了100次时钟中断，如果是，则调用print_ticks()函数，输出“100 ticks”。

定时器中断处理的流程为： 初始化时，先设置sstatus的supervisor中断使能位；clock_set_next_event()函数通过OpenSBI提供的接口set_sbi_timer()，实现每秒100次时钟中断。根据设置，“所有中断都跳到alltraps处理”，触发时钟中断时，trapentry.S中的__alltraps进行保存上下文，并跳转到trap.c的中断处理函数trap()。trap()函数参数为切换前上下文的结构体，调用trap_dispatch函数，将中断异常和异常处理的工作分发给两个函数interrupt_handler和exception_handler。interrupt_handler函数根据中断类型，输出对应文字，设置下一次时钟中断。中断处理结束后，返回trapentry.S，执行__trapret，恢复之前保存的寄存器和状态信息，从S态中断返回到U态。
### （3）扩展练习1
异常处理流程：
1. 当处理器检测到异常或中断时，处理器暂停执行当前指令，跳转到异常和中断处理程序入口地址。
2. 在进入中断处理程序时保存寄存器的值
3. 系统调用对应的处理函数，根据中断的类型执行相应的处理。
4. 处理完成后，恢复保存的寄存器，将控制权交还给被中断的程序。
   
指令mov a0, sp目的是将栈指针 sp的值保存到寄存器 a0 中。这样做的目的是将栈的地址传递给后续的处理函数，让这些函数能够访问保存的寄存器或栈上的数据。

在SAVE_ALL中，寄存器保存到栈中的位置是根据栈指针sp的位置确定的。保存的顺序会在汇编代码中定义，以确保恢复时能够准确还原寄存器的状态。每个寄存器的位置通过对 sp 的偏移量来确定。

需要保存所有寄存器。在 __alltraps中保存所有寄存器是为了保护被中断的程序。对于任何中断，无论是否使用所有寄存器，保存所有通用寄存器的状态可以确保在中断处理程序执行期间不会破坏被中断的程序的状态，这样当中断处理完成后，被中断的程序可以继续正确执行。

### （4）扩展练习2
汇编代码csrw sscratch, sp将栈指针寄存器 sp的值保存到 sscratch 控制状态寄存器中，汇编代码csrrw s0, sscratch, x0读取 sscratch的值并保存到 s0，同时将 x0（即零）写入 sscratch。这两条汇编指令的目的是在中断或异常处理时保存栈指针的值。
保存stval和scause 等寄存器的目的是记录异常或中断的信息，在处理中断或异常时使用。中断处理完成后，这些寄存器的内容失去作用，所以不需要恢复。
### （5）扩展练习3
非法指令异常处理：

cprintf("Exception type:Illegal instruction\n");

cprintf("Illegal instruction caught at 0x%lx\n",tf->epc);

tf->epc += 4;

断点异常处理：

cprintf("Exception type: breakpoint\n");

cprintf("breakpoint caught at 0x%lx\n",tf->epc);

tf->epc += 2 ;

指导书中提到的mret为常规指令，长度为4字节，需要将 tf->epc增加4字节，跳过引发异常的指令。断点指令ebreak为压缩指令，长度是2字节，需要将 tf->epc增加2字节，跳过引发异常的指令。
运行之后发现没有预期的输出，原因是没有引发对应的异常，在init.c里插入异常：

asm volatile("mret");
asm volatile("ebreak");
asm volatile 嵌入了汇编代码，并且告诉编译器不要优化或重新排序。

根据指导书的内容，mret指令用于从M态中断返回到S态或U态，在非M态下执行会引发非法指令异常。ebreak，执行这条指令会触发一个断点中断从而进入中断处理流程。
