这篇文章是这两天看的内容的一个总结，因为内容的难度比较大，所以有很多地方需要以后来纠正。

###内存管理区

因为硬件的限制，所以内核不能对所有的页都同等对待。有些页位于内存中特定的物理地址上，所以不能将其用于一些特定的任务。由于这种限制，所以内核把页分为不同的区（zone）。内核使用区对具有相似特性的页进行分组。

分为：

+ ZONE_DMA。用于DMA操作
+ ZONE_NORMAL。能够正常映射的页
+ ZONE_HIGHMEM。不能永久映射到内核地址空间

硬件限制的原因主要有两个：

+ 一些硬件只能用某些特定的内存地址来执行DMA（直接内存访问）
+ 一些体系结构的内存的物理寻址范围比虚拟寻址范围要大的多。这样一些内存不能永久地映射到内核空间中。

> 关于第二点我的理解是在32位系统中，虚拟地址的范围是4GB，按照3：1的比例，用户空间使用3GB的虚拟地址，内核空间使用1GB的虚拟地址，如果内核空间管理和使用的内存超过1G，那么虚拟地址存在不够用的情况，所以需要按照一定的映射方式进行映射。但是对于64位的系统不存在上面的问题。

下图是对1G的虚拟地址进行划分：


![](img/virtual_address.jpg)

直接映射区域是虚拟地址和物理地址进行一一映射完成的区域，能够直接进行访问。


###Buddy算法
Buddy算法是内核分配一组连续的页框而建立的一种健壮，高效的分配策略。

内存管理的过程中经常会遇到的就是内存碎片，内存碎片分成两种：

+ 外部碎片。随着内存的分配和释放，空闲的空间被分成小片段。当所有的总的可用内存之和可以满足请求，但是并不连续时，就出现了外部碎片问题。
+ 内部碎片。例如需要的空间可能是7K，但是实际分配的过程中是给了8K。然后剩余的空间1K是已经被分配过的，但是使用者根本使用不了的空间。

Buddy算法能够在一定程度上缓解外部碎片问题。算法的主要思想是将空闲内存分成11个大小不同的块链表（链表的元素是该种类块的大小第一页的页描述符），每个块链表分别包含大小为1,2,4,8,16,32,64,128,256,512,1024个连续的页。每个块的第一个页的物理地址是该块大小的整数倍。例如：其实有点类似与字节对齐，**假设物理地址用第0页，第1页，第2页这样的索引表示**，那么大小包含16个页的块，其起始地址为16,32,48这样的位置。这样的对齐方式能够给让分配和合并正常运行。

算法分配页的过程是：如果请求的页面不是2的幂的形式，刚好大于请求页面的2的幂的页面数。例如此时请求为250页，那么分配的是256页。然后从256页的块链表中开始寻找空闲块。如果没有从512中找，找到之后将512分成两个部分，一个部分用于请求，另外一个部分加入256页块链表中。如果512页块链表中也没有，那么从1024页块链表中寻找。如果1024中没有，给出错误信号。如果有，将1024中的256页用于请求，然后将剩余的768页中拿出512页加入到512页块链表中，再把最后的256页插入256页块链表中。

算法释放页的过程就是上面过程的逆过程，也是算法名字的由来。内核试图把大小为b的一对空闲伙伴合并为一个大小为2b的单独块。满足下面的三个条件成为伙伴：

+ 两个块具有相同的大小，计为b
+ 他们的物理地址是连续的
+ 第一块的第一个页的物理地址是2*b的整数倍

算法数据结构图：

![](img/buddy_data_struct.gif)

>其实这个算法还缺少一个初始化的过程，我认为只需要被管理的页面为2^k就行。例如k=6。此时free_area初始话为一个64页的块。这样在分配和释放的过程中既满足了物理地址的对齐要求，也让算法的初始化过程简单。

分配页面的内核部分代码：

```

static struct page *__rmqueue(struct zone *zone, unsigned int order)
{
	struct free_area *area;
    unsigned int current_order;
    /* 遍历free_area 寻找合适的链表块 */
    for (current_order = order; current_order < 11; ++current_order;)
    {
    	area = zone->free_area + current_order;
        if (!list_empty(&area->free_list))
        	goto block_found;
    }
    return NULL;
    block_found:
    /* 获取链表的第一个节点 */
    page = list_entry(area->free_list.next, sturct page, lru);
    /* 从空闲链表块中删除该节点 */
    list_del(&page->lru);
    ClearPagePirvate(page);
    page->private = 0;
    area->nr_free--;
    zone->free_pages -= 1UL << order;

	/* 如果该块页面数大于请求的数量，那么需要将剩余的连续区域重新分配到其他空闲链表块中 */
    size = 1 << curr_order;
    while (curr_order > order)
    {
    	area--;
        cur_order--;
        size >>= 1;
        /* page是响应请求的页面第一页，每次循环将size减半，计算buddy页的起始位置进行重新分配 */
        buddy = page + size;
        /* insert buddy as first element in the list */
        list_add(buddy->lru, &area->free_list);
        area->nr_free__;
        buddy->private = curr_order;
        setPagePrivate(buddy);
    }
    return page;
}
```

释放页面的内核部分代码:

```
__free_pages_bulk(struct page *page, unsigned int order)
{
	struct page * base = zone->zone_mem_map;
	unsigned long buddy_idx, page_idx = page - base;
	struct page * buddy, * coalesced;
	int order_size = 1 << order;

	zone->free_pages += order_size;

	while (order < 10) {
    	/**
         * 这里可能不好理解，一个page_idx的buddy_idx要么在page_idx左侧那么，这时就是page_idx - order_size。order_size是1 << order。或者在右侧，那么就是page_idx + order_size。在左侧还是右侧需要根据page_idx的第order位是0还是1判断，如果是0，那么是page_idx + order_size。如果是1,需要page_idx - order_size
         */
		buddy_idx = page_idx ^ (1 << order);
		buddy = base + buddy_idx;
		if (!page_is_buddy(buddy, order)) //这块我认为少了一个page参数
			break;
		list_del(&buddy->lru);
		zone->free_area[order].nr_free--;
		ClearPagePrivate(buddy);
		buddy->private = 0;
		page_idx &= buddy_idx;
		order++;
	}

    /* 将最后合并的区域添加到空闲块链表 */
    coalesced = base + page_idx;
	coalesced->private = order;
	SetPagePrivate(coalesced);
	list_add(&coalesced->lru, &zone->free_area[order].free_list);
	zone->free_area[order].nr_free++;
}
```


内容来自：


[Linux 伙伴算法简介](https://www.cnblogs.com/cherishui/p/4246133.html)

[高端内存映射之kmap持久内核映射--Linux内存管理(二十)](https://blog.csdn.net/gatieme/article/details/52705142)

《深入理解Linux内核》