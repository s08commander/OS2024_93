##  <center> 实验报告</center>
### <center>小组成员</center>
张涛丹 2212009        
申展2211321        
克梦宇2211765

### 练习1：**分配并初始化一个进程控制块（需要编码）**

alloc_proc 函数（位于 kern/process/proc.c 中）负责分配并返回一个新的 struct proc_struct 结构，用于存储新建立的内核线程的管理信息。ucore 需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。
【提示】在 alloc_proc 函数的实现中，需要初始化的 proc_struct 结构中的成员变量至少包括：
state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name。
请在实验报告中简要说明你的设计实现过程。请回答如下问题：
• 请说明 proc_struct 中 struct context context 和 struct trapframe *tf 成员变量含义和
在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）


在proc.h中，定义结构体proc_struct，用来管理进程信息。代码如下：由提示信息可知：proc_struct结构体中除了list_entry_t list_link和list_entry_t hash_link两个变量之外都需要初始化。

```c
struct proc_struct {
enum proc_state state;                      // 进程状态
int pid;                                    // 进程ID
int runs;                                   // 进程被调度运行的次数
uintptr_t kstack;                           // 进程内核栈指针
volatile bool need_resched;                 // 是否需要重新调度
struct proc_struct *parent;                 // the parent process
struct mm_struct *mm;                       // 进程的内存管理结构
struct context context;                     // 进程的CPU上下文
struct trapframe *tf;                       // 中断帧指针
uintptr_t cr3;                              // 页目录表PDT的基址
uint32_t flags;                             // 进程标志位
char name[PROC_NAME_LEN + 1];               // 进程名称
list_entry_t list_link;                     // 全局进程链表中的节点
list_entry_t hash_link;                     // 哈希链表中的节点
};
...
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
    //LAB4:EXERCISE1 YOUR CODE
        proc->state = PROC_UNINIT;//初始状态为未初始化
        proc->pid = -1;//pid为未赋值
        proc->runs = 0;//运行次数初始为 0
        proc->kstack = 0;//除了idleproc其他线程的内核栈都要后续分配
        proc->need_resched =0; // 不需要立即调度切换线程
        proc->parent = NULL;//没有父线程
        proc->mm = NULL;//未分配内存
        memset(&(proc->context), 0, sizeof(struct context));// 清空上下文
        proc->tf = NULL; //中断帧指针未分配
        proc->cr3 = boot_cr3;//内核线程的cr3为boot_cr3，即页目录为内核页目录表
        proc->flags = 0;// 进程标志位初始化为 0
        memset(proc->name, 0, PROC_NAME_LEN+1);// 进程名清空
    }
    return proc;
}
```

- struct context context：

```c
struct context {
uintptr_t ra;// 返回地址
uintptr_t sp;// 栈指针
uintptr_t s0;// s0-11被调用者保存的寄存器
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

如上所示，struct context 是用于保存和恢复进程运行时 CPU 上下文的结构体，每个进程都包含一个 context 字段，用于保存该进程的上下文信息。

操作系统支持多进程运行时，需要通过调度器在多个进程之间切换。当切换时，当前进程的运行状态（寄存器、栈指针等）需要保存，以便后续能继续执行。在进程切换时，当前进程的寄存器内容会保存到其 context 中，当调度器切换回某个进程时，需要将保存的上下文恢复到 CPU。通过将 context 中保存的寄存器值加载回 CPU，进程可以从切换前的状态继续执行。

- struct trapframe *tf：

```c
struct pushregs {
uintptr_t zero;  // Hard-wired zero
uintptr_t ra;    // Return address
uintptr_t sp;    // Stack pointer
uintptr_t gp;    // Global pointer
uintptr_t tp;    // Thread pointer
uintptr_t t0;    // Temporary
uintptr_t t1;    // Temporary
uintptr_t t2;    // Temporary
uintptr_t s0;    // Saved register/frame pointer
uintptr_t s1;    // Saved register
uintptr_t a0;    // Function argument/return value
uintptr_t a1;    // Function argument/return value
uintptr_t a2;    // Function argument
uintptr_t a3;    // Function argument
uintptr_t a4;    // Function argument
uintptr_t a5;    // Function argument
uintptr_t a6;    // Function argument
uintptr_t a7;    // Function argument
uintptr_t s2;    // Saved register
uintptr_t s3;    // Saved register
uintptr_t s4;    // Saved register
uintptr_t s5;    // Saved register
uintptr_t s6;    // Saved register
uintptr_t s7;    // Saved register
uintptr_t s8;    // Saved register
uintptr_t s9;    // Saved register
uintptr_t s10;   // Saved register
uintptr_t s11;   // Saved register
uintptr_t t3;    // Temporary
uintptr_t t4;    // Temporary
uintptr_t t5;    // Temporary
uintptr_t t6;    // Temporary
};
struct trapframe {
struct pushregs gpr;
uintptr_t status;
uintptr_t epc;
uintptr_t badvaddr;
uintptr_t cause;
};
```

struct trapframe 描述了当发生trap（中断、异常或系统调用）时，保存处理器状态的结构。

- 其结构体中的struct pushregs gpr用来保存通用寄存器的状态，以便处理完成后恢复。
    
    struct pushregs又包括：
    zero：恒定为 0 的寄存器。
    ra：返回地址（函数调用的返回位置）。
    sp：栈指针（指向当前栈的顶端）。
    gp：全局指针（指向全局数据段）。
    tp：线程指针（用于线程本地存储）。
    t0-t6：临时寄存器。
    s0-s11：保存寄存器（需要在函数调用间保持值）。
    a0-a7：函数参数或返回值寄存器。
    
- uintptr_t status用于保存 CPU 的状态寄存器值（如中断使能位、特权级位等）来判断当前上下文是内核态还是用户态。
- uintptr_t epc是异常程序计数器，用来保存发生异常或中断时的程序计数器地址。指示触发陷阱的指令位置，处理完成后可以从该位置继续执行。
- uintptr_t badvaddr用来保存异常地址，例如触发非法内存访问时的地址。在页错误等内存异常中，指示出错的内存地址，便于处理和诊断。
- uintptr_t cause用来保存陷阱的原因代码，指示trap类型，并分发给对应的处理函数。




trapframe与 context 的区别：
**trapframe**：主要用于保存中断、异常或系统调用发生时的上下文，保存的信息较为全面（包括 status、epc 等）。是进程切换中的一部分，但更多用于中断处理。
**context**：保存进程切换时的通用寄存器状态。


### 练习2：**为新创建的内核线程分配资源（需要编码）**

 创建一个内核线程需要分配和设置好很多资源。kernel_thread 函数通过调用 **do_fork** 函数完成具体内核线程的创建工作。do_kernel 函数会调用 alloc_proc 函数来分配并初始化一个进程控制块，但 alloc_proc 只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore 一般通过 do_fork 实际创建新的内核线程。do_fork 的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。因此，我们**实际需要”fork”的东西就是 stack 和 trapframe**。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在 kern/process/proc.c 中的 do_fork 函数中的处理过程。它的大致执行步骤包括：
 
 - 调用 alloc_proc，首先获得一块用户信息块。
 - 为进程分配一个内核栈。
 - 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
 - 复制原进程上下文到新进程
 - 将新进程添加到进程列表
 - 唤醒新进程
 - 返回新进程号
 请在实验报告中简要说明你的设计实现过程。请回答如下问题：
 • 请说明 ucore 是否做到给每个新 fork 的线程一个唯一的 id？请说明你的分析和理由。


1. 完善do_fork函数：
```c
    // 1.调用alloc_proc分配一个proc_struct
    proc = alloc_proc();
    if (proc == NULL) // 如果分配失败
        goto fork_out;  // 返回内存不足错误
    proc->parent = current; // 设置子进程的父进程为当前进程
    // 2.调用setup_kstack为子进程分配一个内核栈
    if (setup_kstack(proc) != 0) 
        goto bad_fork_cleanup_proc; // 如果分配失败，清理已分配的 proc_struct

     // 3. 调用 copy_mm 函数，复制或共享父进程的内存管理信息
    if (copy_mm(clone_flags, proc) != 0) 
        goto bad_fork_cleanup_kstack; // 如果失败，清理已分配的内核栈  

    // 4. 调用copy_thread()函数复制父进程的中断帧和上下文信息到子进程
    copy_thread(proc, stack, tf);

    // 5. 将子进程的proc_struct插入hash_list && proc_list
    bool intr_flag;
    local_intr_save(intr_flag); // 关闭中断，保证操作的原子性
    
    proc->pid = get_pid(); // 为子进程分配唯一的 PID
    hash_proc(proc); //将子进程插入全局哈希表建立映射
    list_add(&proc_list, &(proc->list_link)); // 将子进程插入全局链表
    nr_process ++; // 增加全局进程计数
    
    local_intr_restore(intr_flag); // 恢复中断

    // 6.调用wakeup_proc使新子进程RUNNABLE
    wakeup_proc(proc);

    // 7.使用子进程pid设置获取值
    ret = proc->pid;
  
```
2. ucore可以做到给每个新fork的线程一个唯一的id。ucore在每次创建新的内核进程时，都会调用proc.c中get_pid函数，为新进程分配一个唯一的pid，作为新进程的proc->pid字段。get_pid函数如下：

```c
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}
```

- 函数中首先使用static_assert宏确保MAX_PID大于MAX_PROCESS，保证有足够的PID可用。随后声明相关变量，如next_safe和last_pid，用于跟踪下一个安全的PID以及上一次分配的PID。

- 递增last_pid，如果达到MAX_PID，则将其重置，并跳转到inside。如果last_pid大于等于next_safe（last_pid已经被使用），则将next_safe设置为MAX_PID，进入repeat循环：遍历进程链表，检查当前进程的 ID 是否等于last_pid，如果相等，则递增last_pid。同时，更新next_safe为当前进程中最大的PID值。

- 最后，当找到一个没有被其他进程使用的last_pid时，这个值就被返回作为新分配的PID，因此唯一。

### 练习3：**编写 proc_run 函数（需要编码）**

proc_run 用于将指定的进程切换到 CPU 上运行。它的大致执行步骤包括： 
• 检查要切换的进程是否与当前正在运行的进程相同，如果相同则不需要切换。
• 禁 用 中 断。 你 可 以 使 用/kern/sync/sync.h 中 定 义 好 的 宏 local_intr_save(x) 和
local_intr_restore(x) 来实现关、开中断。
• 切换当前进程为要运行的进程。
• 切换页表，以便使用新进程的地址空间。/libs/riscv.h 中提供了 lcr3(unsigned int cr3)
函数，可实现修改 CR3 寄存器值的功能。
• 实现上下文切换。/kern/process 中已经预先编写好了 switch.S，其中定义了 switch_to() 函
数。可实现两个进程的 context 切换。
• 允许中断。
请回答如下问题：
• 在本实验的执行过程中，创建且运行了几个内核线程？

1.  按实验要求的步骤完成proc_run函数，代码如下
```c
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag; 
        struct proc_struct *prev = current;
        current = proc;
        local_intr_save(intr_flag); // 保持当前中断状态到intr_flag并禁用中断
        {
            // 当前进程设为待调度的进程
            current = proc;
            // 页目录表包含了虚拟地址到物理地址的映射关系,将当前进程的虚拟地址空间映射关系切换为新进程的映射关系.
            // 确保指令和数据的地址转换是基于新进程的页目录表进行的
            // 将当前的cr3寄存器改为需要运行进程的页目录表，其实就是更新页表
            lcr3(current->cr3);
            // 进行上下文切换，保存原线程的寄存器并恢复待调度线程的寄存器
            switch_to(&(prev->context), &(current->context));
        }
        local_intr_restore(intr_flag); // 恢复中断状态
    }
}
```
1. 在本实验的执行过程中，创建且运行了几个内核线程？

通过kernel_thread函数、proc_init函数等的定义可知，本次实验共建立了两个内核线程，分别是idleproc内核线程和initproc内核线程。

- idleproc内核线程是最初的内核线程，其完成内核中各个子线程的创建以及初始化，用于调度其他进程线程，也代表空闲状态下的 CPU 运行。其pid=0，状态为 PROC_RUNNABLE，need_resched=1，表示需要进行调度。
- initproc内核线程是第二个内核线程，通过 kernel_thread 函数创建，会调用init_main函数，该线程主要是为了显示实验成功并打印出字符串"hello world"。



### **扩展练习 Challenge：**

说明语句 local_intr_save(intr_flag);....local_intr_restore(intr_flag); 是如何实现开关中断的？


```c
//定义一个内联函数 __intr_save，用于保存当前中断状态并禁用中断
static inline bool __intr_save(void) {
    if (read_csr(sstatus) & SSTATUS_SIE) {
        // 检查 sstatus 寄存器的 SIE 位是否为 1，表示中断是否启用
        intr_disable();// 如果 SIE 位为 1，则调用 intr_disable 函数禁用中断
        return 1;
    }
    return 0;
}

static inline void __intr_restore(bool flag) {// 用于根据保存的状态恢复中断
    if (flag) {// 检查 flag 参数是否为 true
        intr_enable();// 如果 flag 为 true，则调用 intr_enable 函数重新启用中断
    }
}

#define local_intr_save(x) \  //用于保存当前中断状态并禁用中断
    do {                   \
        x = __intr_save(); \  //调用 __intr_save 函数，将当前中断状态保存到变量 x 中，同时禁用中断
    } while (0)
#define local_intr_restore(x) __intr_restore(x);//定义一个宏 local_intr_restore，根据保存的状态 x 恢复中断

void intr_enable(void) { set_csr(sstatus, SSTATUS_SIE); }

void intr_disable(void) { clear_csr(sstatus, SSTATUS_SIE); }
```

local_intr_save(intr_flag) 和 local_intr_restore(intr_flag) 用于实现中断的开关。
- local_intr_save 会保存当前的中断状态（启用或禁用）到 intr_flag，并禁用中断，确保后续的关键代码段不被中断打断；
- local_intr_restore 则根据保存的状态 intr_flag 恢复中断，使其恢复为原来的状态。这一机制通过操作 RISC-V 的 sstatus 寄存器中的 SIE 位（Supervisor Interrupt Enable）实现，中断开启时设置该位为 1，关闭时清除为 0，从而控制中断的启用与禁用。

###