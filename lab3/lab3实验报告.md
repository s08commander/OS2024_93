##  <center> 实验报告</center>
### <center>小组成员</center>
张涛丹 2212009        
申展2211321        
克梦宇2211765

## （1）练习1



## （2）练习2



## （3）练习3

```c
if(swap_in(mm, addr, &page) == 0){
            if(page_insert(mm->pgdir,page,addr,perm)==0){
                swap_map_swappable(mm,addr,page,1);
                page->pra_vaddr = addr;
            }
```

设计思路：首先调用swap_in 将页面从磁盘加载到内存,然后调用`page_insert 将页面插入页表，然后设置页面为可交换状态，最后更新结构体对象page中的虚拟地址，便于追踪页面在虚拟地址空间中的位置。

#### 1. **页目录项（PDE）和页表项（PTE）对 uCore 实现页替换算法的潜在用处**

**有效性检查**：

PDE 和 PTE 中的有效位（Valid/Present 位）可以表明对应的页是否在内存中。如果缺页发生，硬件会触发缺页异常。

页替换算法可以通过这些位快速判断页面是否需要从磁盘加载或换出。

**权限管理**：

PDE 和 PTE 中的权限位（如可写位 `PTE_W` 和用户权限位 `PTE_U`）可以控制对页面的读写访问。这些信息可以辅助页替换算法判断页面是否可写，从而避免不必要的页面置换。

**页标识**：

PTE 中的物理页帧地址（Physical Page Frame Address）与页表项的逻辑地址共同提供页面的唯一标识。页替换算法通过这些信息维护内存和磁盘之间的页面映射。

**访问记录**：

PTE 中的访问位（Accessed 位）和修改位（Dirty 位）可以反映页面的使用状态。uCore 的页替换算法可以利用这些位决定页面的优先级（例如使用 LRU 或 CLOCK 算法时，访问位尤为重要）

#### 2. **缺页服务例程访问内存时发生页访问异常，硬件要做的事情**

如果缺页服务例程在执行过程中访问内存且发生页访问异常，硬件会进行以下操作：

**触发异常**：

处理器会生成一个页访问异常（Page Fault）中断，将当前执行流切换到异常处理。

**保存上下文**：

硬件会保存引发异常的上下文，包括当前指令地址（`EIP`）和相关寄存器，以便后续恢复。

**异常信息存储**：

处理器会将异常相关信息（例如异常地址、错误代码等）存储到特定寄存器中（如 x86 中的 CR2 寄存器保存访问的虚拟地址）。

**跳转到缺页处理**：

硬件会根据异常向量表，将执行流切换到缺页处理的入口地址。

#### 3. **`Page` 全局变量与页表中 PDE 和 PTE 的关系**

**Page 对象与 PTE**：

每个 `Page` 对象直接对应一个物理页帧。而在页表中，PTE 的物理地址字段会指向对应 `Page` 的物理内存地址。

**Page 对象与 PDE**：

PDE 并不直接对应某个具体的 `Page`，而是指向包含多个 PTE 的页表。页表中的 PTE 通过指向具体的物理页帧，间接与 `Page` 对象相关联。

## （4）练习4
要实现的代码有三部分

**初始化函数**：

```c
static int _clock_init_mm(struct mm_struct *mm) {
    list_init(&pra_list_head);
    curr_ptr = &pra_list_head;
    mm->sm_priv = &pra_list_head;
    return 0;
}
```

调用list_init初始化 pra_list_head，使其成为一个空链表。

设置 curr_ptr指向链表头。

将 `mm->sm_priv` 绑定到链表头，使页替换算法可以通过 `mm` 访问链表。

**页标记函数**：

```
    list_add_before(&pra_list_head, entry);
    page->visited = 1;
```

调用list_add_before将页面插入到链表尾部。

page->visited = 1：标记页面为“已访问”，以便后续替换算法进行处理。

**页替换函数**：

```c
curr_ptr = list_next(curr_ptr);
 if (curr_ptr == head) {
            curr_ptr = list_next(curr_ptr);
            if (curr_ptr == head) {
                *ptr_page = NULL;
                cprintf("curr_ptr %p (list empty)\n", curr_ptr);
                break;
            }
        }
        struct Page *page = le2page(curr_ptr, pra_page_link);
        if (!page->visited) {
            *ptr_page = page;
            list_del(curr_ptr);
            cprintf("curr_ptr %p\n",curr_ptr);
            break;
        } else {
            // 如果页面已被访问，重置访问标志
            page->visited = 0;
        }
```

list_next(curr_ptr)：移动指针到链表的下一个对象。

如果 curr_ptr指向 head，循环回到起点，如果head的下一个对象还是head，说明是空链表

当找到 visited == 0 的页面时，将其从链表中删除并返回。如果visited==1，则置为0。

**clock算法和FIFO算法的不同**：

```c
list_entry_t *head=(list_entry_t*) mm->sm_priv;
    list_entry_t *entry=&(page->pra_page_link);
 
    assert(entry != NULL && head != NULL);
    //record the page access situlation

    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
    list_add(head, entry);
```

FIFO把最新加入的页面放在head之后，这代表最早加入的页面将会被放在链表的最后。

```c
list_entry_t* entry = list_prev(head);
    if (entry != head) {
        list_del(entry);
        *ptr_page = le2page(entry, pra_page_link);
    } else {
        *ptr_page = NULL;
    }
```

上面是FIFO的页替换函数的实现，可以看出如果链表不为空，函数将会删除链表尾部的页面，也就是最早加入的页面，体现了先入先出的思想，

不同之处在于，clock根据页面的访问状态和访问的时间顺序进行替换，会替换最早未被访问的页面。而FIFO只根据页面访问的时间顺序进行替换，会替换最早访问的页面。

