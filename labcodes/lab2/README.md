# Lab 2

## 练习 1：实现 first-fit 连续物理内存分配算法

首先，ucore内存管理的启动是通过 `init.c` 中的 `pmm_init();` 调用，该函数会通过 `init_pmm_manager();` 将内存管理器初始化为 `default_pmm_manager`，所以我们只需要编写 `default_pmm_manager` 中的各个函数就能实现特定的内存管理机制。我们需要注意到，Lab2在 `bootasm.S` 中增加了一段内存探测的内容，即从 `probe_memory` 到 `finish_probe` 的部分，这部分代码会从 `0x8000` 位置开始保存内存分布信息，也就是填充  `memlayout.h` 里定义的结构体 `e820map` ，该结构体会在 `pmm.c` 中的 `page_init` 函数中使用。总结来说，[系统执行中地址映射分成了三个阶段](https://learningos.github.io/ucore_os_webdocs/lab2/lab2_3_3_5_4_maping_relations.html)：

- 第一阶段是从 bootloader 的 `start` 函数到执行 ucore kernel 的 `kern_entry` 函数之前，这一阶段跟Lab1中一样，采取的是对等映射。此时：

  ```
  lab2 stage 1: virt addr = linear addr = phy addr
  ```

- 第二阶段是从 `kern_entry` 函数到执行 `pmm_init` 函数之前，这一阶段首先创建了初始页目录表，并开启了分页模式。此时，内核所处的实际物理地址仍然是低地址，我们在保留低虚拟地址空间(0\~4MB)对等映射的同时，将等长的高虚拟地址空间(0xC0000000\~0xC0000000+4MB)也映射到对应的低物理地址空间(0\~4MB)。然后我们就能跳转到高地址空间，并清除原来低虚拟地址空间(0\~4MB)的对等映射，以腾出低虚拟地址空间给用户程序。通过查看 entry.S，我们可以发现，实现上述映射的途径是建立对应的页目录和页表。在设置页目录的时候，我们实际把虚拟地址空间(0\~4MB)和(0xC0000000\~0xC0000000+4MB)的页目录项都映射到了同一个页表：

  ```assembly
  # kernel builtin pgdir
  # an initial page directory (Page Directory Table, PDT)
  # These page directory table and page table can be reused!
  .section .data.pgdir
  .align PGSIZE
  __boot_pgdir:
  .globl __boot_pgdir
      # map va 0 ~ 4M to pa 0 ~ 4M (temporary)
      .long REALLOC(__boot_pt1) + (PTE_P | PTE_U | PTE_W)
      .space (KERNBASE >> PGSHIFT >> 10 << 2) - (. - __boot_pgdir) # pad to PDE of KERNBASE
      # map va KERNBASE + (0 ~ 4M) to pa 0 ~ 4M
      .long REALLOC(__boot_pt1) + (PTE_P | PTE_U | PTE_W)
      .space PGSIZE - (. - __boot_pgdir) # pad to PGSIZE
  
  .set i, 0
  __boot_pt1:
  .rept 1024
      .long i * PGSIZE + (PTE_P | PTE_W)
      .set i, i + 1
  .endr
  ```

  离开这个阶段的时候，实际只保留了从高虚拟地址空间(0xC0000000\~0xC0000000+4MB)到低物理地址空间(0\~4MB)的映射：

  ```
  lab2 stage 2: virt addr = linear addr = phy addr + 0xC0000000
  ```

- 第三阶段是从 `pmm_init` 函数被调用开始，该函数会将页目录表项补充完成（从 0~4M 扩充到 0~KMEMSIZE）。除此以外，由于在bootloader中设置的段表只包括了内核态的代码段和数据段描述符，该函数会完善其他段描述符，包括用户态的代码段和数据段描述符以及 TSS（段）的描述符。此时：

  ```
  lab2 stage 3: virt addr = linear addr = phy addr + 0xC0000000
  ```

第二，要注意ucore中双向循环链表的实现方式跟一般数据结构课程中描述翻方式不同。由于ucore中多种数据结构都有建立双向循环链表的需要，为了避免针对每种特定的数据结构都实现单独的链表类型，ucore创建了 `list_entry` 类型，其他类型通过将该类型作为成员变量的方式实现双向链表。具体的内容查看实验指导书的 [双向循环链表](https://learningos.github.io/ucore_os_webdocs/lab0/lab0_2_6_2_1_linked_list.html) ，尤其是要理解我们是通过 `le2xxx` 的方式访问链表节点所在的宿主数据结构的。

```c
struct list_entry {
    struct list_entry *prev, *next;
};
```

第三，我们需要弄清楚ucore中空闲内存是如何表示的。ucore使用了双向循环链表将各个空闲内存区域串联起来，该循环链表或者其头结点是 `free_area` ，其他链表结点是各空闲内存区域的head page，只不过使用的是像上面提到的用head page的 `Page` 结构体中的 `page_link` 来相互串联的。因此，我们要根据这种设计的要求在分配和释放内存的时候对各个数据结构做相应的更改。

```c
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```

下面，我们具体来看 default_pmm.c 中的相关函数，理解它们现有实现的功能并且判断是否需要根据first fit算法的要求做更改：

- 首先是对 `free_area` 进行初始化的函数，这个不需要做任何处理：

  ```c
  static void
  default_init(void) {
      list_init(&free_list);
      nr_free = 0;
  }
  ```

- 然后是 `default_init_memmap` ，这个函数接受 `pmm.c` 中 `page_init` 函数的调用，而 `page_init` 函数会根据之前BIOS阶段探测并存储到结构体 `e820map` 中的内存信息对物理内存对应的所有 `Page` 结构进行初始化。如果内存探测显示某段内存区域是可用的( `type == E820_ARM` )，则将该段内存区域上下取整过后，以该区域首个物理页对应的 `Page` 结构地址和物理页的个数作为参数调用 `default_init_memmap` 。`default_init_memmap` 首先对这些 `Page` 做统一的初始化，然后对head page对应的 `Page` 做特殊处理，最后递增 `free_area` 中空闲页的计数 `nr_free` 并将该段区域加入 `free_area` 链表。这里，实验指导书和实际代码中对 `PG_property` 的解释不同，如果按照实验指导书的说法，我们需要把每个空闲内存 `Page` 都 `SetPageProperty` 。但是这里我们采取实际代码仓库给出的解释，只对空闲内存区域的第一个 `Page` 做 `SetPageProperty` ，也就不需要做任何更改。

  ```c
  #define PG_property                 1       
  // if this bit=1: the Page is the head page of a free memory block(contains some continuous_addrress pages), and can be used in alloc_pages; if this bit=0: if the Page is the the head page of a free memory block, then this Page and the memory block is alloced. Or this Page isn't the head page.
  ```
  
  ```c
  static void
  default_init_memmap(struct Page *base, size_t n) {
      assert(n > 0);
      struct Page *p = base;
      for (; p != base + n; p ++) {
          assert(PageReserved(p));
          p->flags = p->property = 0;
          set_page_ref(p, 0);
      }
      base->property = n;
    SetPageProperty(base);
      nr_free += n;
      list_add(&free_list, &(base->page_link));
  }
  ```
  
- 然后是 `default_alloc_pages` 函数，该函数根据参数按序查找 `free_area` ，寻求符合大小要求的空闲内存块并分配。具体来说，第一步先从头开始遍历空闲内存双向链表，直到找到空间 `>= n` 的可分配区域，这部分工作示例代码已经给出了。第二步是对相关的各个数据结构做修改，包括将多余部分空闲内存作为新的节点插入双向链表以及从 `free_area` 中删除已分配的内存页计数并删除代表原有空闲内存的节点。这部分的工作示例代码并没有正确完成，需要我们自己实现。

  ```c
  static struct Page *
  default_alloc_pages(size_t n) {
      assert(n > 0);
      if (n > nr_free) {
          return NULL;
      }
      struct Page *page = NULL;
      list_entry_t *le = &free_list;
      while ((le = list_next(le)) != &free_list) {
          struct Page *p = le2page(le, page_link);
          if (p->property >= n) {
              page = p;
              break;
          }
      }
      if (page != NULL) {
          if (page->property > n) {
              struct Page *p = page + n;
              p->property = page->property - n;
              SetPageProperty(p);
              list_add(&(page->page_link), &(p->page_link));
          }
          list_del(&(page->page_link));
          nr_free -= n;
          ClearPageProperty(page);
      }
      return page;
  }
  ```

- 然后是 `default_free_pages` 函数，该函数是 `default_alloc_pages` 的反过程。给出的示例代码已经实现了大部分功能，甚至已经帮我们做了可能的合并，只不过我们需要自己找到合适的位置插入新释放的内存区域。

  ```c
  static void
  default_free_pages(struct Page *base, size_t n) {
      assert(n > 0);
      struct Page *p = base;
      for (; p != base + n; p ++) {
          assert(!PageReserved(p) && !PageProperty(p));
          p->flags = 0;
          set_page_ref(p, 0);
      }
      base->property = n;
      SetPageProperty(base);
      list_entry_t *le = list_next(&free_list);
      while (le != &free_list) {
          p = le2page(le, page_link);
          le = list_next(le);
          if (base + base->property == p) {
              base->property += p->property;
              ClearPageProperty(p);
              list_del(&(p->page_link));
          }
          else if (p + p->property == base) {
              p->property += base->property;
              ClearPageProperty(base);
              base = p;
              list_del(&(p->page_link));
          }
      }
      le = &free_list; 
      while ((le = list_next(le)) != &free_list)
          if ((base + base->property) < le2page(le, page_link))
              break;
      nr_free += n;
      list_add_before(le, &base->page_link);
  }
  ```


在正确完成上述任务后，当你再次运行 `make qemu` 的时候会看到如下输出，表明我们需要完成下一个练习了：

```
kernel panic at kern/mm/pmm.c:478:
    assertion failed: get_pte(boot_pgdir, PGSIZE, 0) == ptep
```

## 练习 2：实现寻找虚拟地址对应的页表项

练习2的注释已经将要做什么以及可能用到的工具都给出来了，我们只需要按照帮助说明编写代码就行了。我们先尝试性地获取要返回的 `ptep` ，然后检查检查对应的页目录项是否有效，如果有效就直接返回。如果无效，我们看调用时是否指定了创建选项 `create` ，当未指定时直接返回。如果指定了 `create` ，我们就通过练习1中的页分配函数获取存储页表的物理页，并将该物理页的索引置1，同时根据该页表的地址设置页目录项，最后返回相应的页表项地址。相对来说，需要注意的是部分细节，比如虚拟地址和物理地址的转换、不同数据类型之间的转换等。

```c
    int32_t pdindex = PDX(la);
    pde_t *pdep = pgdir + pdindex;
    int32_t ptindex = PTX(la);
    pte_t *ptep = (pte_t *) KADDR(*pdep & 0xFFFFF000) + ptindex;
    if (*pdep & PTE_P) return ptep;
    if (!create) 
        return NULL;
    else {
        struct Page * pt = alloc_page();
        if (pt == NULL) return NULL;
        set_page_ref(pt, 1);
        uintptr_t newpt = KADDR(page2pa(pt));
        memset(newpt, 0, PGSIZE);
        *pdep = page2pa(pt) | PTE_USER;
        return newpt + ptindex;
    }
```

> 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中每个组成部分的含义以及对 ucore 而言的潜在用处。

![image-20200917172941108](https://raw.githubusercontent.com/shawnxbc/PicGo/master/technotes/image-20200917172941108.png)

> 如果 ucore 执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

不知道为什么Lab2的练习里会出现这个问题，实际上相关描述在[Lab3 Page Fault异常处理](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab3/lab3_4_page_fault_handler.html)，具体内容查阅该部分即可。

## 练习 3：释放某虚地址所在的页并取消对应二级页表项的映射

这个练习完全就是翻译代码模板中给出的提示，基本没有任何难度。不过，我最开始写完发现通过不了测试，检查了一下是因为我给 `pte2page` 传的指针，因为我看到代码说明里是 `struct Page *page pte2page(*ptep)` ，实际需要传 `pte` 。

```c
    if (*ptep & PTE_P) {
        struct Page *p = pte2page(*ptep);
        page_ref_dec(p);
        if (p->ref == 0)
            free_page(p);
        *ptep = 0;
        tlb_invalidate(pgdir, la);
    }
```

到这里应该就能通过全部测试了，`make qemu` 可以看到输出：

```
check_alloc_page() succeeded!
check_pgdir() succeeded!
check_boot_pgdir() succeeded!
```

>数据结构 Page 的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

由于页目录项中存储着页表的物理基址，而页表项中存储着页的物理基址，所以显然是跟 `Page` 数组中的某项有对应关系的，它们之间的转换可以通过 `pa2page` 和 `page2pa` 实现。

> 如果希望虚拟地址与物理地址相等，则需要如何修改 lab2，完成此事？

- 将 `KERNBASE` 修改为 `0x00000000` ；
- 将链接脚本的内核加载机制修改为 `0x00100000` ；
- 之前我们定义了两个页目录项，分别是低虚拟地址空间(0\~4MB)和高虚拟地址空间(KERNBASE\~KERNBASE+4MB)。由于原来 `KERNBASE` 是 `0xC0000000` ，其对应的页目录项在靠后的位置，所以设置低虚拟地址空间映射后需要"pad to PDE of KERNBASE"。现在 `KERNBASE` 改成了 `0x00000000` ，我们就需要删除 `entry.S` 中原来与低虚拟地址0~4M对等映射对应的map和unmap以及相应的"pad to PDE of KERNBASE"命令。
- 最后 `check_pgdir();` 和 `check_boot_pgdir();` 会因为内存布局的改变而无法通过，所以要么对它们 进行修改或者直接注释掉。

修改过后查看运行结果，发现目标内存映射确实实现了：

```
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0x0010002f (phys)
  etext  0x00105f25 (phys)
  edata  0x0011c000 (phys)
  end    0x0011cf28 (phys)
Kernel executable memory footprint: 116KB
memory management: default_pmm_manager
e820map:
  memory: 0009fc00, [00000000, 0009fbff], type = 1.
  memory: 00000400, [0009fc00, 0009ffff], type = 2.
  memory: 00010000, [000f0000, 000fffff], type = 2.
  memory: 07ee0000, [00100000, 07fdffff], type = 1.
  memory: 00020000, [07fe0000, 07ffffff], type = 2.
  memory: 00040000, [fffc0000, ffffffff], type = 2.
check_alloc_page() succeeded!
-------------------- BEGIN --------------------
PDE(0e0) 00000000-38000000 38000000 urw
  |-- PTE(38000) 00000000-38000000 38000000 -rw
PDE(001) fac00000-fb000000 00400000 -rw
  |-- PTE(000e0) fac00000-face0000 000e0000 urw
  |-- PTE(00001) fafeb000-fafec000 00001000 -rw
--------------------- END ---------------------
++ setup timer interrupts
100 ticks
100 ticks
...
```

