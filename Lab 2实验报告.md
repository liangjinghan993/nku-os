***

### 练习1：理解first-fit 连续物理内存分配算法（思考题）

### first-fit 连续物理内存分配算法作为物理内存分配一个很基础的方法，需要同学们理解它的实现过程。请大家仔细阅读实验手册的教程并结合`kern/mm/default_pmm.c`中的相关代码，认真分析default\_init，default\_init\_memmap，default\_alloc\_pages， default\_free\_pages等相关函数，并描述程序在进行物理内存分配的过程以及各个函数的作用。 请在实验报告中简要说明你的设计实现过程。请回答如下问题：

### 你的first fit算法是否有进一步的改进空间？

1.  `default_init` 函数：

    *   初始化空闲内存块链表 `free_list` 为一个空链表。
    *   初始化空闲内存块的数量 `nr_free` 为0。
2.  `default_init_memmap` 函数：

    *   用于初始化一个自由块（有给定的 `addr_base` 地址和 `n` 页数量）。
    *   初始化每一页（在 `memlayout.h` 中定义）的属性，包括：

        *   设置 `p->flags` 的 `PG_property` 标志，表示该页有效。
        *   如果该页是自由的并且不是自由块的第一页，将 `p->property` 设置为0。
        *   如果该页是自由的并且是自由块的第一页，将 `p->property` 设置为自由块中页的总数。
        *   将 `p->ref` 设置为0，因为该页是自由的，没有引用。
        *   使用 `p->page_link` 将该页链接到 `free_list` 链表中。
    *   更新 `nr_free` 以记录自由内存块的总数。
3.  `default_alloc_pages` 函数：

    *   用于分配一定数量的内存页面（`n` 个页面）。
    *   遍历空闲内存块链表 `free_list`，寻找第一个大于或等于 `n` 的空闲块。
    *   如果找到满足需求的页面块，从链表中移除该块，并将其返回。
    *   如果分配的页面块大小大于请求的大小，将多余的部分分割为一个新的页面块，然后将其添加到链表中。
    *   更新 `nr_free` 以反映新的自由内存块数量。
4.  `default_free_pages` 函数：

    *   用于释放一定数量的内存页面。
    *   根据释放的页面块的基地址，查找在 `free_list` 中的正确位置，并将这些页面块插入。
    *   重新设置页面块的各个字段，例如 `p->ref` 和 `p->flags`。
    *   尝试合并相邻的自由内存块，以减少内存碎片。
5.  `default_nr_free_pages` 函数：

    *   返回当前可用的自由页面数量，即 `nr_free` 的值。
6.  `basic_check` 函数：

    *   执行一些基本的分配和释放检查以验证内存管理的正确性和一些其他功能。
    *   分配页面，释放页面，检查页面引用计数，检查页面是否合法等。
7.  `default_check` 函数：

    *   用于检查First-Fit分配算法的正确性和性能。
    *   检查自由内存块链表的属性，确保它们被正确维护。
    *   执行一些分配和释放操作以验证算法的正确性。
    *   进行性能检查以评估算法的性能和效率。&#x20;

        first-fit算法的优化方案如下:

    1.在 `default_init_memmap` 函数中优化寻找插入位置的算法，使用二分查找算法。
    2.在`default_alloc_pages`函数中优化寻找空闲页的算法，使用二分查找算法。
    3.在`default_free_pages`函数中记录前后页的信息，在删除空闲页时更高效地查找相邻页。

***

### 练习2：实现 Best-Fit 连续物理内存分配算法（需要编程）

### 在完成练习一后，参考`kern/mm/default_pmm.c`对First Fit算法的实现，编程实现Best Fit页面分配算法，算法的时空复杂度不做要求，能通过测试即可。 请在实验报告中简要说明你的设计实现过程，阐述代码是如何对物理内存进行分配和释放，并回答如下问题：

### 你的 Best-Fit 算法是否有进一步的改进空间？

`kern/mm/best_`*`fit_`*`pmm.c`代码：

```c
#include <pmm.h>
#include <list.h>
#include <string.h>
#include <best_fit_pmm.h>
#include <stdio.h>
free_area_t free_area;

#define free_list (free_area.free_list)
#define nr_free (free_area.nr_free)

static void
best_fit_init(void) {
    list_init(&free_list);
    nr_free = 0;
}

static void
best_fit_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));

        /*LAB2 EXERCISE 2: 2112086*/ 
        // 清空当前页框的标志和属性信息，并将页框的引用计数设置为0
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    if (list_empty(&free_list)) {
        list_add(&free_list, &(base->page_link));
    } else {
        list_entry_t* le = &free_list;
        while ((le = list_next(le)) != &free_list) {
            struct Page* page = le2page(le, page_link);
             /*LAB2 EXERCISE 2: 2112086*/ 
            // 编写代码
            // 1、当base < page时，找到第一个大于base的页，将base插入到它前面，并退出循环
            // 2、当list_next(le) == &free_list时，若已经到达链表结尾，将base插入到链表尾部
            if (base < page) {
                list_add_before(le, &(base->page_link));
                break;
            } else if (list_next(le) == &free_list) {
                list_add(le, &(base->page_link));
            }
        }
    }
}

static struct Page *
best_fit_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    size_t min_size = nr_free + 1;
     /*LAB2 EXERCISE 2: 2112086*/ 
    // 下面的代码是first-fit的部分代码，请修改下面的代码改为best-fit
    // 遍历空闲链表，查找满足需求的空闲页框
    // 如果找到满足需求的页面，记录该页面以及当前找到的最小连续空闲页框数量
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n && p->property < min_size) {
            page = p;
            min_size = p->property;
        }
    }

    if (page != NULL) {
        list_entry_t* prev = list_prev(&(page->page_link));
        list_del(&(page->page_link));
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;
            SetPageProperty(p);
            list_add(prev, &(p->page_link));
        }
        nr_free -= n;
        ClearPageProperty(page);
    }
    return page;
}

static void
best_fit_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));
        p->flags = 0;
        set_page_ref(p, 0);
    }
    /*LAB2 EXERCISE 2: 2112086*/ 
    // 编写代码
    // 具体来说就是设置当前页块的属性为释放的页块数、并将当前页块标记为已分配状态、最后增加nr_free的值
    base->property = n;
    SetPageProperty(base);
    nr_free += n;

    if (list_empty(&free_list)) {
        list_add(&free_list, &(base->page_link));
    } else {
        list_entry_t* le = &free_list;
        while ((le = list_next(le)) != &free_list) {
            struct Page* page = le2page(le, page_link);
            if (base < page) {
                list_add_before(le, &(base->page_link));
                break;
            } else if (list_next(le) == &free_list) {
                list_add(le, &(base->page_link));
            }
        }
    }

    list_entry_t* le = list_prev(&(base->page_link));
    if (le != &free_list) {
        p = le2page(le, page_link);
        /*LAB2 EXERCISE 2: YOUR 2112086*/ 
         // 编写代码
        // 1、判断前面的空闲页块是否与当前页块是连续的，如果是连续的，则将当前页块合并到前面的空闲页块中
        // 2、首先更新前一个空闲页块的大小，加上当前页块的大小
        // 3、清除当前页块的属性标记，表示不再是空闲页块
        // 4、从链表中删除当前页块
        // 5、将指针指向前一个空闲页块，以便继续检查合并后的连续空闲页块
        if (p + p->property == base) {
            p->property += base->property;
            ClearPageProperty(base);
            list_del(&(base->page_link));
            base = p;
        }
    }

    le = list_next(&(base->page_link));
    if (le != &free_list) {
        p = le2page(le, page_link);
        if (base + base->property == p) {
            base->property += p->property;
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
    }
}

static size_t
best_fit_nr_free_pages(void) {
    return nr_free;
}

static void
basic_check(void) {
    struct Page *p0, *p1, *p2;
    p0 = p1 = p2 = NULL;
    assert((p0 = alloc_page()) != NULL);
    assert((p1 = alloc_page()) != NULL);
    assert((p2 = alloc_page()) != NULL);

    assert(p0 != p1 && p0 != p2 && p1 != p2);
    assert(page_ref(p0) == 0 && page_ref(p1) == 0 && page_ref(p2) == 0);

    assert(page2pa(p0) < npage * PGSIZE);
    assert(page2pa(p1) < npage * PGSIZE);
    assert(page2pa(p2) < npage * PGSIZE);

    list_entry_t free_list_store = free_list;
    list_init(&free_list);
    assert(list_empty(&free_list));

    unsigned int nr_free_store = nr_free;
    nr_free = 0;

    assert(alloc_page() == NULL);

    free_page(p0);
    free_page(p1);
    free_page(p2);
    assert(nr_free == 3);

    assert((p0 = alloc_page()) != NULL);
    assert((p1 = alloc_page()) != NULL);
    assert((p2 = alloc_page()) != NULL);

    assert(alloc_page() == NULL);

    free_page(p0);
    assert(!list_empty(&free_list));

    struct Page *p;
    assert((p = alloc_page()) == p0);
    assert(alloc_page() == NULL);

    assert(nr_free == 0);
    free_list = free_list_store;
    nr_free = nr_free_store;

    free_page(p);
    free_page(p1);
    free_page(p2);
}

// LAB2: below code is used to check the best fit allocation algorithm 
// NOTICE: You SHOULD NOT CHANGE basic_check, default_check functions!
static void
best_fit_check(void) {
    int score = 0 ,sumscore = 6;
    int count = 0, total = 0;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        assert(PageProperty(p));
        count ++, total += p->property;
    }
    assert(total == nr_free_pages());

    basic_check();

    #ifdef ucore_test
    score += 1;
    cprintf("grading: %d / %d points\n",score, sumscore);
    #endif
    struct Page *p0 = alloc_pages(5), *p1, *p2;
    assert(p0 != NULL);
    assert(!PageProperty(p0));

    #ifdef ucore_test
    score += 1;
    cprintf("grading: %d / %d points\n",score, sumscore);
    #endif
    list_entry_t free_list_store = free_list;
    list_init(&free_list);
    assert(list_empty(&free_list));
    assert(alloc_page() == NULL);

    #ifdef ucore_test
    score += 1;
    cprintf("grading: %d / %d points\n",score, sumscore);
    #endif
    unsigned int nr_free_store = nr_free;
    nr_free = 0;

    // * - - * -
    free_pages(p0 + 1, 2);
    free_pages(p0 + 4, 1);
    assert(alloc_pages(4) == NULL);
    assert(PageProperty(p0 + 1) && p0[1].property == 2);
    // * - - * *
    assert((p1 = alloc_pages(1)) != NULL);
    assert(alloc_pages(2) != NULL);      // best fit feature
    assert(p0 + 4 == p1);

    #ifdef ucore_test
    score += 1;
    cprintf("grading: %d / %d points\n",score, sumscore);
    #endif
    p2 = p0 + 1;
    free_pages(p0, 5);
    assert((p0 = alloc_pages(5)) != NULL);
    assert(alloc_page() == NULL);

    #ifdef ucore_test
    score += 1;
    cprintf("grading: %d / %d points\n",score, sumscore);
    #endif
    assert(nr_free == 0);
    nr_free = nr_free_store;

    free_list = free_list_store;
    free_pages(p0, 5);

    le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        count --, total -= p->property;
    }
    assert(count == 0);
    assert(total == 0);
    #ifdef ucore_test
    score += 1;
    cprintf("grading: %d / %d points\n",score, sumscore);
    #endif
}
//这个结构体在
const struct pmm_manager best_fit_pmm_manager = {
    .name = "best_fit_pmm_manager",
    .init = best_fit_init,
    .init_memmap = best_fit_init_memmap,
    .alloc_pages = best_fit_alloc_pages,
    .free_pages = best_fit_free_pages,
    .nr_free_pages = best_fit_nr_free_pages,
    .check = best_fit_check,
};

```

截图展示：

make qemu：

![image.png](https://gitee.com/liang-jinghan888/nku-operating-system-2023/raw/master/%E5%9B%BE%E7%89%87%E6%96%87%E4%BB%B6%E5%A4%B9/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-10-20%20142858.png)

make grade：

![image.png](https://gitee.com/liang-jinghan888/nku-operating-system-2023/raw/master/%E5%9B%BE%E7%89%87%E6%96%87%E4%BB%B6%E5%A4%B9/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-10-20%20142950.png)

实现过程简述：

这个Best-Fit算法的实现过程包括初始化、内存分配、内存释放、合并和性能检查等步骤。算法的核心思想是选择最小的合适内存块来满足分配请求，以减少内存碎片。然后，它通过维护一个有序的空闲内存块链表来管理可用的内存块。这使得分配和释放内存块的操作更加高效和准确。

1.  `best_fit_init` 函数：

    *   初始化空闲内存块链表 `free_list` 为一个空链表。
    *   初始化空闲内存块的数量 `nr_free` 为0。
2.  `best_fit_init_memmap` 函数：

    *   为给定的一段内存块标记页面属性，并将它们插入到空闲内存块链表中。
    *   如果 `base` 处的内存块的属性标记为保留（PageReserved），则清空标志和属性信息，将引用计数设置为0，然后添加到空闲内存块链表。
    *   更新 `nr_free` 表示可用的空闲页面数量。
    *   如果空闲内存块链表为空，则将 `base` 添加到链表中。
    *   如果链表不为空，通过遍历链表找到 `base` 应该插入的位置，以维持链表的有序性。
3.  `best_fit_alloc_pages` 函数：

    *   用于分配一定数量的内存页面（`n` 个页面）。
    *   遍历空闲内存块链表，查找满足需求的空闲页面块，选择最小的可以容纳 `n` 个页面的页面块。
    *   如果找到满足需求的页面块，将其从链表中移除并分配给请求者。
    *   如果被分配的页面块大小大于请求的大小，将多余的部分拆分为一个新的页面块，然后将其添加到链表中。
    *   更新 `nr_free` 表示可用的空闲页面数量。
4.  `best_fit_free_pages` 函数：

    *   用于释放一定数量的内存页面。
    *   将释放的页面块的属性标记为已释放，增加 `nr_free` 计数。
    *   如果可能，合并相邻的空闲页面块，以减少内存碎片。
5.  `best_fit_nr_free_pages` 函数：

    *   返回当前可用的空闲页面数量，即 `nr_free` 的值。
6.  `best_fit_check` 函数：

    *   用于检查最佳适应算法的正确性和性能。
    *   检查空闲内存块链表的属性，确保它们被正确维护。
    *   执行基本的分配和释放检查以确保算法的正确性。
    *   进行性能检查以评估算法的性能和效率。

best-fit算法进一步的改进空间：

1.  **优化空闲空间搜索**：当前的代码遍历整个空闲列表以找到最佳适应的块。当存在许多空闲块时，这种线性搜索可能很慢。可以通过保持空闲列表按块大小排序来进行优化，以便快速找到最佳适应的块，而无需遍历整个列表。
2.  **内存碎片化**：最佳适应可能导致内存碎片化，因为它可能会从较大的空闲块中分配较小的块。可以实施内存整理或碎片整理技术，以最小化碎片化并保持较大的空闲块可用。
3.  **缓存**：为了减少查找最佳适应块的搜索时间，可以实现缓存机制来记住上次分配发生的位置。这样，可以从上次分配的块开始搜索自由块，因为它可能在大小上接近。
4.  **分配时合并**：在分配块时，检查是否可以将邻近的空闲块合并以创建较大的空闲块。这有助于随着时间的推移减少碎片化。
5.  **使用更高效的数据结构**：根据特定的用例，可以使用更高效的数据结构，如二叉树或堆，来管理空闲内存块。这些数据结构可以提供更快的自由块插入和删除。

***

### 扩展练习Challenge1：buddy system（伙伴系统）分配算法（需要编程）

### Buddy System算法把系统中的可用存储空间划分为存储块(Block)来进行管理, 每个存储块的大小必须是2的n次幂(Pow(2, n)), 即1, 2, 4, 8, 16, 32, 64, 128...

### 参考[伙伴分配器的一个极简实现](https://gitee.com/link?target=http%3A%2F%2Fcoolshell.cn%2Farticles%2F10427.html)， 在ucore中实现buddy system分配算法，要求有比较充分的测试用例说明实现的正确性，需要有设计文档。

1.1 伙伴系统算法

伙伴算法（Buddy system）把所有的空闲页框分为11个块链表，每块链表中分布包含特定的连续页框地址空间，比如第0个块链表包含大小为2^0  ^个连续的页框，第1个块链表中，每个链表元素包含2个页框大小的连续地址空间，….，第10个块链表中，每个链表元素代表4M的连续地址空间。每个链表中元素的个数在系统初始化时决定，在执行过程中，动态变化。伙伴算法每次只能分配2的幂次页的空间，比如一次分配1页，2页，4页，8页，…，1024页等等，每页大小一般为4K，因此，伙伴算法最多一次能够分配4M的内存空间。

![image.png](https://gitee.com/liang-jinghan888/nku-operating-system-2023/raw/master/%E5%9B%BE%E7%89%87%E6%96%87%E4%BB%B6%E5%A4%B9/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-10-20%20143032.png)

![](https://gitee.com/liang-jinghan888/nku-operating-system-2023/raw/master/%E5%9B%BE%E7%89%87%E6%96%87%E4%BB%B6%E5%A4%B9/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-10-20%20143055.png)



1.2 内存分配

伙伴分配器分配和释放内存的基本单位是阶（order），分配n阶页块的过程如下：

1.  查看是否有空闲的n阶页块， 如果有， 直接分配，如果没有，进行下一步
2.  查看是否有n+1阶空闲页块，如果有， 将其拆为两个n阶页块，一个分配出去，一个添加到n阶页块中管理； 如果没有，则继续查找更高阶的页块

释放n阶页块时， 查看它的伙伴是否空间， 如果不空闲， 则将其释放到n阶页块中；如果空闲，则一起组成n+1 阶页块， 释放到n+1阶页块中进行管理。

1.2.1 分配函数

分配函数为alloc\_pages，最终调用到\_\_alloc\_pages\_nodemask

![](https://gitee.com/liang-jinghan888/nku-operating-system-2023/raw/master/%E5%9B%BE%E7%89%87%E6%96%87%E4%BB%B6%E5%A4%B9/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-10-20%20143119.png)

    struct page *
    __alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
    			struct zonelist *zonelist, nodemask_t *nodemask)
    {
    	struct zoneref *preferred_zoneref;
    	struct page *page = NULL;
    	unsigned int cpuset_mems_cookie;
    	int alloc_flags = ALLOC_WMARK_LOW|ALLOC_CPUSET|ALLOC_FAIR;
    	gfp_t alloc_mask; /* The gfp_t that was actually used for allocation */
    	struct alloc_context ac = {
                    /*根据gfp_mask,找到合适的zone idx和migrate类型 */
    		.high_zoneidx = gfp_zone(gfp_mask),
    		.nodemask = nodemask,
    		.migratetype = gfpflags_to_migratetype(gfp_mask),
    	};
     
    retry_cpuset:
    	cpuset_mems_cookie = read_mems_allowed_begin();
     
    	/* We set it here, as __alloc_pages_slowpath might have changed it */
    	ac.zonelist = zonelist;
     
    	/* Dirty zone balancing only done in the fast path */
    	ac.spread_dirty_pages = (gfp_mask & __GFP_WRITE);
     
    	/*根据分配掩码,确认先从哪个zone分配 */
    　　　　　　　/* The preferred zone is used for statistics later */
    	preferred_zoneref = first_zones_zonelist(ac.zonelist, ac.high_zoneidx,
    				ac.nodemask ? : &cpuset_current_mems_allowed,
    				&ac.preferred_zone);
    	if (!ac.preferred_zone)
    		goto out;
    	ac.classzone_idx = zonelist_zone_idx(preferred_zoneref);
     
    	/* First allocation attempt */
    	alloc_mask = gfp_mask|__GFP_HARDWALL;
    	/*快速分配 */
    	page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
    	if (unlikely(!page)) {
    		/*慢速分配，涉及到内存回收，暂不分析 */
    		page = __alloc_pages_slowpath(alloc_mask, order, &ac);
    	}
     
    out:
    	if (unlikely(!page && read_mems_allowed_retry(cpuset_mems_cookie)))
    		goto retry_cpuset;
     
    	return page;
    }

1.2.2 单个页面的分配

get\_page\_from\_freelist->buffered\_rmqueue:

order=0时，单个页面直接从per cpu的free list分配，这样效率最高

```
if (likely(order == 0)) {
		struct per_cpu_pages *pcp;
		struct list_head *list;
		local_irq_save(flags);
		pcp = &this_cpu_ptr(zone->pageset)->pcp;
		list = &pcp->lists[migratetype];
		/*如果对应list为空，则从伙伴系统拿内存*/
		if (list_empty(list)) {
			pcp->count += rmqueue_bulk(zone, 0,
					pcp->batch, list,
					migratetype, gfp_flags);
			if (unlikely(list_empty(list)))
				goto failed;
		}
 
		/*分配一个页面 */
　　　　　　　　　　　　　　　　if ((gfp_flags & __GFP_COLD) != 0)
			page = list_entry(list->prev, struct page, lru);
		else
			page = list_entry(list->next, struct page, lru);
 
		if (!(gfp_flags & __GFP_CMA) &&
				is_migrate_cma(get_pcppage_migratetype(page))) {
			page = NULL;
			local_irq_restore(flags);
		} else {
			list_del(&page->lru);
			pcp->count--;
		}
	}

```

1.2.3 多个页面的分配（order>1）

get\_page\_from\_freelist->buffered\_rmqueue->\_\_rmqueue:

    static struct page *__rmqueue(struct zone *zone, unsigned int order,
    				int migratetype, gfp_t gfp_flags)
    {
    	struct page *page = NULL;
            /*CMA内存的分配 */
    	if ((migratetype == MIGRATE_MOVABLE) && (gfp_flags & __GFP_CMA))
    		page = __rmqueue_cma_fallback(zone, order);
     
    	if (!page)/*根据order和migrate type找到对应的freelist分配内存 */
    		page = __rmqueue_smallest(zone, order, migratetype);
     
    	if (unlikely(!page))/*当对应的migrate type无法满足order分配时，进行fallback规则分配
    ，分配规则定义在fallbacks数组中。 */
    		page = __rmqueue_fallback(zone, order, migratetype);
     

    	return page;
    }

这里分析migrate type能够满足内存分配的情况

    static inline
    struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
                            int migratetype)
    {
        unsigned int current_order;
        struct free_area *area;
        struct page *page;

        /* 根据order，逐级向上查找*/
        for (current_order = order; current_order < MAX_ORDER; ++current_order) {
            area = &(zone->free_area[current_order]);
            if (list_empty(&area->free_list[migratetype]))
                continue;

            page = list_entry(area->free_list[migratetype].next,
                                struct page, lru);
            list_del(&page->lru);
            rmv_page_order(page);
            area->nr_free--;

    　　/* 切蛋糕,如果current_order大于目标order,则要把多余的内存挂到对应的order链表.*/
            expand(zone, page, order, current_order, area, migratetype);
            set_pcppage_migratetype(page, migratetype);
            return page;
        }

        return NULL;
    }
    static inline void expand(struct zone *zone, struct page *page,
    	int low, int high, struct free_area *area,
    	int migratetype)
    {
    	unsigned long size = 1 << high;//先一分为2
     
    	while (high > low) {
    		area--;/* 回退到下一级链表*/
    		high--;
    		size >>= 1;/*Size减半 */
                   　/*接入下一级空闲链表 */
    		list_add(&page[size].lru, &area->free_list[migratetype]);
    		area->nr_free++;
    		set_page_order(&page[size], high);
    	}
    }

1.3 内存释放

页面的释放，最终调用到\_\_free\_one\_page函数.

首先两块内存是伙伴块,必须满足以下条件:

1.  伙伴不能在空洞页面中，要有实实在在的物理页面/the buddy is not in a hole/
2.  伙伴块在伙伴系统中，也就是伙伴块要是空闲的，没有被分配出去的内存块/the buddy is in the buddy system/
3.  要有相同的order /a page and its buddy have the same order/
4.  位于同一个zone./a page and its buddy are in the same zone/
5.  物理上要相连.且两个page的起始pfn号一定相差2^order， ^则有计算公式B2 = B1 ^ (1 << O)

    static inline unsigned long
    \_\_find\_buddy\_index(unsigned long page\_idx, unsigned int order)
    {
    return page\_idx ^ (1 << order);//计算buddy page的index
    }
    static inline void \_\_free\_one\_page(struct page \*page,
    unsigned long pfn,
    struct zone \*zone, unsigned int order,
    int migratetype)
    {
    unsigned long page\_idx;
    unsigned long combined\_idx;
    unsigned long uninitialized\_var(buddy\_idx);
    struct page \*buddy;
    unsigned int max\_order;

        max_order = min_t(unsigned int, MAX_ORDER, pageblock_order + 1);

        /*计算page_idx, */
        page_idx = pfn & ((1 << MAX_ORDER) - 1);

        VM_BUG_ON_PAGE(page_idx & ((1 << order) - 1), page);
        VM_BUG_ON_PAGE(bad_range(zone, page), page);

    continue\_merging:
    while (order < max\_order - 1) {
    /\*找到buddy ix \*/
    buddy\_idx = \_\_find\_buddy\_index(page\_idx, order);
    buddy = page + (buddy\_idx - page\_idx);
    /\*判断是否是伙伴块 */
    if (!page\_is\_buddy(page, buddy, order))
    goto done\_merging;
    /*
    \* Our buddy is free or it is CONFIG\_DEBUG\_PAGEALLOC guard page,
    \* merge with it and move up one order.
    \*/
    if (page\_is\_guard(buddy)) {
    clear\_page\_guard(zone, buddy, order, migratetype);
    } else {
    list\_del(\&buddy->lru);
    zone->free\_area\[order].nr\_free--;
    rmv\_page\_order(buddy);
    }
    　　　　　　　　　　　　　　　 /\*继续向上合并 \*/
    combined\_idx = buddy\_idx & page\_idx;
    page = page + (combined\_idx - page\_idx);
    page\_idx = combined\_idx;
    order++;
    }

    done\_merging:
    set\_page\_order(page, order);

        /*
         * If this is not the largest possible page, check if the buddy
         * of the next-highest order is free. If it is, it's possible
         * that pages are being freed that will coalesce soon. In case,
         * that is happening, add the free page to the tail of the list
         * so it's less likely to be used soon and more likely to be merged
         * as a higher order page
         */
        if ((order < MAX_ORDER-2) && pfn_valid_within(page_to_pfn(buddy))) {
        	struct page *higher_page, *higher_buddy;
                /*这里检查更上一级是否存在伙伴关系，如果是的,则把page添加到链表末尾，这样有利于页面回收. */
        	combined_idx = buddy_idx & page_idx;
        	higher_page = page + (combined_idx - page_idx);
        	buddy_idx = __find_buddy_index(combined_idx, order + 1);
        	higher_buddy = higher_page + (buddy_idx - combined_idx);
        	if (page_is_buddy(higher_page, higher_buddy, order + 1)) {
        		list_add_tail(&page->lru,
        			&zone->free_area[order].free_list[migratetype]);
        		goto out;
        	}
        }
        /*链接到对应的free list 链表 */
        list_add(&page->lru, &zone->free_area[order].free_list[migratetype]);

    out:
    zone->free\_area\[order].nr\_free++;
    }

1.4 分配器

通过一个数组形式的完全二叉树来监控管理内存，二叉树的节点用于标记相应内存块的使用状态，高层节点对应大的块，低层节点对应小的块，在分配和释放中我们就通过这些节点的标记属性来进行块的分离合并。假设总大小为16单位的内存，我们就建立一个深度为5的满二叉树，根节点从数组下标\[0]开始，监控大小16的块；它的左右孩子节点下标\[12]，监控大小8的块；第三层节点下标\[3-6]监控大小4的块……依此类推

*   在`pmm.c`中在Page数组后为分配器预留内存
*   在`buddy_pmm.c`中实现分配、释放算法
*   在`pmm.c`中将pmm\_manager绑定到该buddy\_pmm\_manager中，实现策略的选择

buddy.h

    #ifndef __BUDDY_H__
    #define __BUDDY_H__

    #include <defs.h>

    struct buddy2 {
    	uint32_t size;
    	uint32_t longest[0];
    };

    extern struct buddy2 *self;

    #endif /* !__BUDDY_H__ */

buddy\_pmm.c

```
#include <pmm.h>
#include <list.h>
#include <string.h>
#include <default_pmm.h>
#include <stdio.h>
#include "buddy.h"
#include "buddy_pmm.h"


//free_area_t free_area;
//unsigned int nr_free;
struct Page *m_base;
int nr_free;
#define LEFT_LEAF(index) (2 * index + 1)
#define RIGHT_LEAF(index) (2 * index + 2)
#define PARENT(index) ((index - 1) / 2)
#define MAX(a,b) (a < b ? b : a)

//#define free_list (free_area.free_list)
//#define nr_free (free_area.nr_free)

int is_power_of_2(int size)
{
	while((size % 2 == 0)) size /= 2;
	return size == 1;
}

static void
buddy_init(void) {
    //list_init(&free_list);
	nr_free = 0; 	   
}

static void
buddy_init_memmap(struct Page *base, size_t n) {
	assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
		p->flags = 0;
        set_page_ref(p, 0);
    }
	
	size_t num = 1;
	while(num <= n) num *= 2;
	num /= 2;

	num = 1024;	//测试时缩小节点数

    nr_free = num;
	m_base = base;
    
	assert(num >= 1 && is_power_of_2(num));
	self->size = num;
	unsigned node_size = num * 2;
	for(int i = 0; i < 2 * num - 1; ++i) {
		if(is_power_of_2(i + 1))
			node_size /= 2;
		self->longest[i] = node_size;
	}
}

static struct Page *
buddy_alloc_pages(size_t size) {
	unsigned index = 0;
	unsigned node_size;
	unsigned offset = 0;
    if (size > nr_free || self == NULL) {
        return NULL;
    }
	if(size <= 0)
		size = 1;
	else if(!is_power_of_2(size))
	{
		unsigned int n = 1;
		while(n < size) n *= 2;
		size = n;
	}
	if(self->longest[index] < size)
		return NULL;
	nr_free -= size;
	for(node_size = self->size; node_size != size; node_size /= 2) {
		if(self->longest[LEFT_LEAF(index)] >= size)
			index = LEFT_LEAF(index);
		else
			index = RIGHT_LEAF(index);
	}
	self->longest[index] = 0;
	offset = (index + 1) * node_size- self->size;
	while (index) {
		index = PARENT(index);
		self->longest[index] = MAX(self->longest[LEFT_LEAF(index)], self->longest[RIGHT_LEAF(index)]);
	}
	return m_base + offset;
}

static void
buddy_free_pages(struct Page *base, size_t n) {
	struct Page *p = base;
	for (; p != base + n; p ++) {
		p->flags = 0;
		set_page_ref(p, 0);
	}
    unsigned offset = base - m_base;
	unsigned node_size, index = 0;
	unsigned left_longest, right_longest;
	assert(self && offset >= 0 && offset < self->size);
	node_size = 1;
	index = offset + self->size - 1;
	for (; self->longest[index] ; index = PARENT(index)) {
		node_size *= 2;
		if (index == 0)
			return;
	}
	self->longest[index] = node_size;
	nr_free += node_size;
	while (index) {
		index = PARENT(index);
		node_size *= 2;
		left_longest = self->longest[LEFT_LEAF(index)];
		right_longest = self->longest[RIGHT_LEAF(index)];
		if (left_longest + right_longest == node_size)
			self->longest[index] = node_size;
		else
			self->longest[index] = MAX(left_longest, right_longest);
	}
}

static size_t
buddy_nr_free_pages(void) {
    return nr_free;
}

void print_node(int index)
{
	cprintf("node%d: %d\n",index,self->longest[index]);
}

void print_from_to(int start, int end)
{
	for(int i = start; i <= end; i++)
		print_node(i);
	cprintf("\n");
}

static void
basic_check(void) {
    
}

// LAB2: below code is used to check the first fit allocation algorithm
// NOTICE: You SHOULD NOT CHANGE basic_check, default_check functions!
static void
buddy_check(void) {
	cprintf("\nBuddy pmm manager!\n\n");
	cprintf("all %d pages!\n",nr_free);
	struct Page *p0, *p1, *p2, *p3;
	p0 = p1 = p2 = p3 = NULL;
	assert((p0 = alloc_pages(70)) != NULL);
	assert((p1 = alloc_pages(35)) != NULL);
	assert((p2 = alloc_pages(257)) != NULL);
	assert((p3 = alloc_pages(63)) != NULL);
	print_from_to(0,30);	
	
	free_page(p1);
	free_page(p3);
	print_from_to(0,30);

	free_page(p0);
	print_from_to(0,30);

	struct Page *p4, *p5;
	p4 = p5 = NULL;
	assert((p4 = alloc_pages(255)) != NULL);
	assert((p5 = alloc_pages(255)) != NULL);
	print_from_to(0,30);

	free_page(p2);
	free_page(p4);
	free_page(p5);
	print_from_to(0,30);
}
//这个结构体在
const struct pmm_manager buddy_pmm_manager = {
    .name = "buddy_pmm_manager",
    .init = buddy_init,
    .init_memmap = buddy_init_memmap,
    .alloc_pages = buddy_alloc_pages,
    .free_pages = buddy_free_pages,
    .nr_free_pages = buddy_nr_free_pages,
    .check = buddy_check,
};

```

buddy\_pmm.h

    #ifndef __KERN_MM_BUDDY_PMM_H__
    #define  __KERN_MM_BUDDY_PMM_H__

    #include <pmm.h>

    extern const struct pmm_manager buddy_pmm_manager;

    #endif /* ! __KERN_MM_BUDDY_PMM_H__ */

![image.png](https://gitee.com/liang-jinghan888/nku-operating-system-2023/raw/master/%E5%9B%BE%E7%89%87%E6%96%87%E4%BB%B6%E5%A4%B9/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-10-20%20143151.png)

***

### 扩展练习Challenge：任意大小的内存单元slub分配算法（需要编程），实现两层架构的高效内存单元分配，第一层是基于页大小的内存分配，第二层是在第一层基础上实现基于任意大小的内存分配。可简化实现，能够体现其主体思想即可。

### 参考[linux的slub分配算法/](https://gitee.com/link?target=http%3A%2F%2Fwww.ibm.com%2Fdeveloperworks%2Fcn%2Flinux%2Fl-cn-slub%2F)，在ucore中实现slub分配算法。要求有比较充分的测试用例说明实现的正确性，需要有设计文档。

***

### 扩展练习Challenge3：硬件的可用物理内存范围的获取方法（思考题）

### 如果 OS 无法提前知道当前硬件的可用物理内存范围，请问你有何办法让 OS 获取可用物理内存范围？

如果操作系统无法提前知道当前硬件的可用物理内存范围，可以尝试使用以下方法让操作系统获取可用物理内存范围：

1.  使用BIOS或UEFI提供的接口：BIOS或UEFI固件通常提供了一些接口或数据结构，可以让操作系统获取硬件的相关信息，包括可用物理内存范围。你可以查阅硬件厂商提供的文档，了解如何使用这些接口或数据结构来获取可用物理内存范围。
2.  使用操作系统提供的API：大多数操作系统提供了API来获取系统的硬件信息，包括可用物理内存范围。例如，在Windows操作系统中，可以使用GetSystemInfo函数来获取系统信息，其中包括可用物理内存范围。在Linux操作系统中，可以使用/proc文件系统中的相关文件来获取系统信息，如/proc/meminfo文件可以提供可用物理内存的信息。
3.  使用第三方库或工具：有些第三方库或工具可以帮助获取系统硬件信息，包括可用物理内存范围。例如，对于C++开发，可以使用库如hwloc、cpuid等来获取硬件信息。对于命令行工具，如dmidecode、lshw等也可以提供硬件信息。OS原理重要知识点

### 本实验重要知识点

1，对物理内存的探测方法。

2，具体的连续物理内存分配算法，包括first\_fit、best\_fit等一系列算法

3，在C语言中使用面向对象思想实现物理内存管理。

### OS重要知识点

1，内存页管理机制：内存页管理机制是操作系统用来管理物理内存和虚拟内存之间映射关系的一种机制。它将物理内存划分为固定大小的页（通常是4KB），并为每个进程维护一个虚拟内存空间，将虚拟内存划分为相同大小的页框。

2，连续物理内存管理：连续物理内存管理是指操作系统将物理内存划分为一系列连续的内存块，并为进程分配这些连续的物理内存块来存储其代码、数据和堆栈等信息。主要包括内存分区、分配和释放内存、碎片管理和内存保护几个步骤。

3，物理内存的管理方法：主要有连续内存管理、分页内存管理、分段内存管理、断页式内存管理和反向映射等

4，页面分配机制：页面分配机制是一种内存管理技术，用于将进程的虚拟地址空间映射到物理内存中的页面。页面分配机制通常与分页内存管理方式结合使用。页面分配机制的主要目标是将进程所需的页面从虚拟内存空间映射到物理内存中。
