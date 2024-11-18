##  <center> 实验报告</center>
### <center>小组成员</center>
张涛丹 2212009        
申展2211321        
克梦宇2211765

## （1）练习1 理解基于FIFO的页面替换算法
**一个页面从被换入到被换出的过程如下：**

#### 换入:

1.alloc_page()（/kern/mm/pmm.c中）：为被换入的页面分配一个空闲的物理页面。它通过调用具体的 pmm_manager->alloc_pages 来在内存中分配物理页

2.get_pte()：根据虚拟地址 addr 查找该地址对应 PTE，此时不需要创建页表项

3.swapfs_read()：从磁盘的交换空间读取页面内容，并将数据加载到分配的物理页面中

#### 当内存不足, 需要换出页面时:

1.alloc_pages()： 如果内存不足, 它调用 swap_out 来调用页面置换算法

2.swap_out()：FIFO算法实现的核心，它调用 sm->swap_out_victim()函数指针，指向FIFO页面置换算法的 swap_out_victim 函数。_fifo_swap_out_victim将从内存中选择一个页面进行换出，FIFO策略的实现靠“替换链表头元素的前一个元素来”实现。这是因为在FIFO的数据结构中，当在_fifo_init_mm()中完成初始化时，将mm->sm_priv 设置为指向链表头部 pra_list_head. 而每次新增一个可交换页面时, 即在 _fifo_map_swappable() 中, 会将进入内存的页面加入链表的头部 list_add(head, entry)。

因此在 swap_out_victim() 中, 根据 FIFO 思想: 最先进入内存的页面将最先被替换, 首先获取 FIFO 链表的头部, 然后使用 list_prev(head) 返回双向链表中头元素的前一个元素, 即链表的最后一个元素, 即"最早进入的页面"。接下来，使用 list_del(entry) 从链表中删除受害者页面节点, 再通过 le2page(entry, pra_page_link) 宏将链表节点 entry 转换回其对应的 Page 结构体, 得到 Page 后即可进行后续操作, 完成物理页的查找和释放。不断换出页面直到 alloc_pages 能成功分配足够数量的物理页.

3.获取 FIFO 链表头部 pra_list_head

4.宏 list_prev(head)： 返回双向链表中头元素的前一个元素, 即链表的最后一个元素

5.宏 list_del(entry) ：从链表中删除受害者页面节点

6.宏 le2page(entry, pra_page_link) 宏：将链表节点 entry 转换回其对应的 Page 结构体并返回

7.得到要换出的页的虚拟地址 v 后, get_pte(mm->pgdir, v, 0) 获取该地址在页表中的页表项

8.swapfs_write() ：将被替换页面的内容写入交换文件

成功: free_page() 释放对应物理页面

失败: map_swappable() 将该页面标记为可换出页面，是指向 _fifo_map_swappable的函数指针, 将可换出页面添加到链表头部.


## （2）练习2:深入理解不同分页模式的工作原理
#### 解释get_pte()中代码的相似性：
```c
pde_t *pdep1 = &pgdir[PDX1(la)];
if (!(*pdep1 & PTE_V)) {
    struct Page *page;
    if (!create || (page = alloc_page()) == NULL) {
        return NULL;
    }
    set_page_ref(page, 1);
    uintptr_t pa = page2pa(page);
    memset(KADDR(pa), 0, PGSIZE);
    *pdep1 = pte_create(page2ppn(page), PTE_U | PTE_V);
}

pde_t *pdep0 = &((pde_t *)KADDR(PDE_ADDR(*pdep1)))[PDX0(la)];
if (!(*pdep0 & PTE_V)) {
    struct Page *page;
    if (!create || (page = alloc_page()) == NULL) {
        return NULL;
    }
    set_page_ref(page, 1);
    uintptr_t pa = page2pa(page);
    memset(KADDR(pa), 0, PGSIZE);
    *pdep0 = pte_create(page2ppn(page), PTE_U | PTE_V);
}
```

分页机制的主要差异在于页表层数和线性地址的解析方式，对于SV32、SV39和SV48这三种分页方式，它们的主要区别在于SV32有两级页表，SV39有三级页表，SV48有四级页表，但是它们共享相同的基本架构，每一级的页表结构和操作逻辑相同：都是检查当前页表项是否有效，无效时分配新的物理页并初始化。因此可以看到get_pte()中分别对两级页表pdep1和pdep0执行相同的操作。在多级递归过程中，也是由上一级指向下一级页表的，因此每一级的初始化逻辑也是类似的。

#### 拆开/合并的价值：
目前get_pte()函数将页表项的查找和页表项的分配合并在一个函数里，优点是可以在一次函数调用中完成页表项查找和分配，它们在逻辑上是高度相关的，这样做减少了一部分重复代码，比如在页表项无效时可以直接分配内存，这样避免了额外的函数调用开销。如果将find_pte和alloc_pte拆分的话，代码的独立性会提升，更利于复杂场景下的使用。总而言之，如果大部分情况下查抄、分配是同时需要执行的，那么当前的方法更优；如果存在很多只查找不分配的场景，那么拆分开来会更合理一些。

## （3）练习3:给未被映射的地址映射上物理页

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

## （4）练习4:补充完成Clock页替换算法
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

## （5）练习五：阅读代码和实现手册，理解页表映射方式相关知识
**如果我们采用”一个大页“ 的页表映射方式，相比分级页表**
#### 好处、优势：
使用单级页表直接映射整个地址空间，最大的优点是简单。具体表现为结构简单：不需要多级迭代，为不同层次的页表分配内存空间，组织页表解构和维护起来很简单；查找、更新简单：直接使用虚拟地址作为索引，没有页表层次，地址转换快，查找和更新都很容易。
#### 坏处、风险：
一个大页表覆盖整个虚拟地址空间，会造成大量的内存浪费：比如对于32位地址空间，需要维护约10^6页表项，尤其是对于那些稀疏的地址空间，许多虚拟地址没有真正使用，但是却为它们保留了许多页表项。使用分级页表不仅能通过页表层级实现对稀疏地址空间存储的优化，还能通过多级扩展提供更大规模地址空间的需求，而非像单一页表那样一次性分配固定大小的地址空间。

此外，大页表的TLB访问效率其实很低，并且如果要满足大规模的存储需求，它对硬件的要求就会比多级页表更高，因为需要更大的页表项cache来支持大规模大页表。

事实上，多级页表的出现就是为了对单一页表在使用时出现的“占用内存过多”现象·进行优化的，通过创造多级层次结构，将多级页表构造为“页表的页表”，就能有效地对页表实现压缩操作并节省内存。
