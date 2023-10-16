# lab2

## 练习1：理解first-fit 连续物理内存分配算法

### first-fit 连续物理内存分配算法作为物理内存分配一个很基础的方法，需要同学们理解它的实现过程。请大家仔细阅读实验手册的教程并结合kern/mm/default_pmm.c中的相关代码，认真分析default_init，default_init_memmap，default_alloc_pages， default_free_pages等相关函数，并描述程序在进行物理内存分配的过程以及各个函数的作用。 请在实验报告中简要说明你的设计实现过程。

default_init函数如下：

![](https://pic.imgdb.cn/item/652d2736c458853aefce28cb.png)

该函数初始化了一个双向链表free_list和一个整型变量nr_free；其中链表free_list用于管理可用内存块，可以追踪哪些内存块当前是可用的，以及它们的位置和大小；而nr_free则表示当前可用的内存块数量，具有计数作用；

```
static void
default_init(void) {
    list_init(&free_list); //调用list_init初始化一个双向链表，链表用来管理可用内存块
    nr_free = 0;//整型变量nr_free用于计数当前可用的内存块数量，初始化为0表示初始不存在空闲内存//块，方便后续内存管理进行正确分配和释放
}
```

在本实验物理内存分配过程中，函数default_init会被调用，进而初始化一个用于管理可用内存块的双向链表free_list和记录可用内存块数量的整型变量nr_free，并且在后续的内存管理中，free_list会动态更新而nr_free也会随当前可用内存块数量而调整；

default_init_memmap函数如下：

![](https://pic.imgdb.cn/item/652d291ec458853aefd4715a.png)

为代码添加注释如下：
```
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0); // 检查页数是否大于0
    
    struct Page *p = base; // 初始化指向基础页的指针p，从基础页开始遍历
    for (; p != base + n; p ++) {
        assert(PageReserved(p)); // 断言该页是保留页，确保代码正确性
        
        p->flags = p->property = 0; // 将该页的标志位和属性信息置为0，表示清空
        set_page_ref(p, 0); // 将该页的引用计数设置为0，表示该页未被引用
    }
    
    base->property = n; // 设置基础页的连续保留页数量为n
    SetPageProperty(base); // 标记基础页为保留页
    
    nr_free += n; // 增加可用的物理页数量
    
    if (list_empty(&free_list)) {
        list_add(&free_list, &(base->page_link)); // 如果free_list为空，则将基础页添加到free_list中
    } else {
        list_entry_t* le = &free_list;
        while ((le = list_next(le)) != &free_list) {
            struct Page* page = le2page(le, page_link); // 获取当前节点对应的Page结构体
            
            if (base < page) { // 如果基础页的地址小于当前节点的地址，将基础页插入到当前节点之前
                list_add_before(le, &(base->page_link));
                break;
            } else if (list_next(le) == &free_list) { // 如果遍历到链表末尾，将基础页插入到链表末尾
                list_add(le, &(base->page_link));
            }
        }
    }
}
```

在物理内存分配过程中，函数default_init_memmap初始化一块连续的保留页，并将其添加到可用物理页的链表中，同时更新相应的计数和标志位。此外该函数还通过断言确保了保留页的正确性和有序性

default_alloc_pages函数如下：

![](https://pic.imgdb.cn/item/652d291ec458853aefd471ae.png)

为函数添加注释如下

```
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0); // 断言检查页数是否大于0
    
    if (n > nr_free) { // 如果需要分配的页数大于可用物理页数量，返回NULL
        return NULL;
    }
    
    struct Page *page = NULL; // 初始化页指针page
    list_entry_t *le = &free_list; // 初始化链表头指针le
    
    while ((le = list_next(le)) != &free_list) { // 遍历可用物理页链表
        struct Page *p = le2page(le, page_link); // 获取当前节点对应的Page结构体
        
        if (p->property >= n) { // 如果当前节点对应的保留页数量大于等于需要分配的页数
            page = p; // 将页指针page指向当前节点对应的Page结构体
            break;
        }
    }
    
    if (page != NULL) { // 如果找到了可用的物理页
        list_entry_t* prev = list_prev(&(page->page_link)); // 获取前一个节点的指针
        list_del(&(page->page_link)); // 从可用物理页链表中删除该页
        
        if (page->property > n) { // 如果找到的页比需要分配的页数多
            struct Page *p = page + n; // 初始化一个指针p，指向页page+n
            p->property = page->property - n; // 计算新页的保留页数量
            SetPageProperty(p); // 标记新页为保留页
            list_add(prev, &(p->page_link)); // 将新页添加到链表中
        }
        
        nr_free -= n; // 减少可用物理页数量
        ClearPageProperty(page); // 标记该页为非保留页
    }
    
    return page; // 返回找到的可用物理页指针，或者NULL
}

```

default_alloc_pages函数在物理内存分配过程中，根据请求从可用物理页链表中寻找连续的物理页进行分配，并返回一个指向其Page结构体的指针，或者返回NULL，并更新内存管理数据结构；

default_free_pages函数如下：

![](https://pic.imgdb.cn/item/652d291ec458853aefd47223.png)

为函数添加注释如下

```
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0); // 检查页数是否大于0
    
    struct Page *p = base; // 初始化一个Page指针p，指向参数base
    for (; p != base + n; p ++) { // 遍历要释放的物理页范围
        assert(!PageReserved(p) && !PageProperty(p)); // 检查物理页是否为保留页和预留页
        p->flags = 0; // 将物理页的flags字段清零
        set_page_ref(p, 0); // 将物理页的引用计数设置为0
    }
    
    base->property = n; // 设置参数base对应的物理页的保留页数量为n
    SetPageProperty(base); // 标记该物理页为保留页
    nr_free += n; // 增加可用物理页数量
    
    if (list_empty(&free_list)) { // 如果可用物理页链表为空
        list_add(&free_list, &(base->page_link)); // 将参数base插入链表头部
    } else { // 如果可用物理页链表不为空
        list_entry_t* le = &free_list; // 初始化链表头指针le
        
        while ((le = list_next(le)) != &free_list) { // 遍历可用物理页链表
            struct Page* page = le2page(le, page_link); // 获取当前节点对应的Page结构体
            
            if (base < page) { // 如果参数base的地址小于当前节点对应的地址
                list_add_before(le, &(base->page_link)); // 在当前节点之前插入参数base
                break;
            } else if (list_next(le) == &free_list) { // 如果已经到达链表尾部
                list_add(le, &(base->page_link)); // 将参数base添加到链表尾部
            }
        }
    }
    
    list_entry_t* le = list_prev(&(base->page_link)); // 获取参数base的前一个节点的指针le
    if (le != &free_list) { // 如果前一个节点存在
        p = le2page(le, page_link); // 将前一个节点转换为Page结构体
        if (p + p->property == base) { // 如果前一个节点和参数base可以合并成一个连续的物理页
            p->property += base->property; // 合并两个连续物理页的保留页数量
            ClearPageProperty(base); // 清除参数base的保留页标记
            list_del(&(base->page_link)); // 从链表中删除参数base
            base = p; // 将参数base更新为合并后的物理页
        }
    }
    
    le = list_next(&(base->page_link)); // 获取参数base的后一个节点的指针le
    if (le != &free_list) { // 如果后一个节点存在
        p = le2page(le, page_link); // 将后一个节点转换为Page结构体
        if (base + base->property == p) { // 如果参数base和后一个节点可以合并成一个连续的物理页
            base->property += p->property; // 合并两个连续物理页的保留页数量
            ClearPageProperty(p); // 清除后一个节点的保留页标记
            list_del(&(p->page_link)); // 从链表中删除后一个节点
        }
    }
}
```

default_free_pages函数的功能是在物理内存释放过程中，将一段连续的物理页归还给可用物理页链表，并更新内存管理数据结构


### 你的first fit算法是否有进一步的改进空间？

First Fit算法是一种简单高效且常用的内存分配算法，选择第一个足够容纳请求大小的空闲块进行分配即可，但会引入碎片问题，其中碎片是指已分配内存与可用内存之间的空闲内存块，具体如下：

1. **内存碎片**：使用First Fit算法可能导致内存碎片的产生，即已分配的内存块之间存在很小的空闲空间，无法满足大内存请求。这会降低内存的利用率；
2. **分配速度**：First Fit算法需要遍历整个空闲内存块链表以找到第一个符合要求的空闲块，随着空闲块数量的增加，分配速度可能变慢。

为改进First Fit算法，可以从以下方面考虑：

1. **Best Fit算法**：最佳适应算法选择最小合适的空闲块进行分配，能更好地利用内存空间并减少碎片。但是它的分配速度相对较慢，需要遍历整个空闲块链表；
2. **分区合并**：在释放内存块时，可以考虑合并相邻的空闲块，以减少碎片。合并相邻空闲块可以通过维护空闲块的链表，并进行合并操作来实现；
3. **动态分区调整**：可以根据程序的运行情况，动态调整内存分区的大小，以适应不同大小的内存请求。这需要综合考虑可用内存块的大小、请求的大小和分配策略。


## 练习2：实现 Best-Fit 连续物理内存分配算法 

### 在完成练习一后，参考kern/mm/default_pmm.c对First Fit算法的实现，编程实现Best Fit页面分配算法，算法的时空复杂度不做要求，能通过测试即可。 请在实验报告中简要说明你的设计实现过程，阐述代码是如何对物理内存进行分配和释放，并回答如下问题：你的 Best-Fit 算法是否有进一步的改进空间？

函数best_fit_init_memmap中编程部分

![](https://pic.imgdb.cn/item/652d30b9c458853aefecd428.png)

![](https://pic.imgdb.cn/item/652d291ec458853aefd47275.png)

![](https://pic.imgdb.cn/item/652d291ec458853aefd472fb.png)

函数best_fit_alloc_pages中编程部分：

![](https://pic.imgdb.cn/item/652d2925c458853aefd48a90.png)

在这一步中实现了best-fit算法，与First-fist算法的区别如下

![](https://pic.imgdb.cn/item/652d2925c458853aefd48aee.png)

其一，条件修改，在First-fist算法仅需要p->property >= n即可，也就是当前内存块能够满足需求即可；而best-fit算法增添条件p->property < min_size，意为不仅要当前内存块能够满足，并且该内存块是最小能够满足需求；

其二，修改了代码逻辑，First-fist算法找到第一个满足需求的内存块之后就会break跳出循环；而best-fit算法需要遍历所有内存块，并不断更新，直至找到最小满足内存块。

函数best_fit_free_pages中的编程部分

![](https://pic.imgdb.cn/item/652d2925c458853aefd48b32.png)

![](https://pic.imgdb.cn/item/652d2931c458853aefd4b46a.png)

## 扩展练习Challenge：硬件的可用物理内存范围的获取方法

### 如果 OS 无法提前知道当前硬件的可用物理内存范围，请问你有何办法让 OS 获取可用物理内存范围？

答：可以借助硬件

·BIOS/UEFI：许多计算机在启动阶段会执行基本输入输出系统（BIOS）或统一的可扩展固件接口（UEFI），这些固件通常具有获取硬件信息的功能，通过调用适当的BIOS或UEFI函数，操作系统可以获取到可用物理内存范围。

·ACPI（高级配置和电源接口）：ACPI是一种标准化的电源管理和配置接口，旨在提供操作系统对硬件的控制和访问，OS可以通过解析ACPI数据表来获取有关硬件的详细信息，包括可用物理内存范围。