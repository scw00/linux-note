# vm_phys_init

前16M 为DMA 地址，前512M为低内存地址。前4g为DMA32内存


# static struct vm_freelist vm_phys_free_queues
    
```
static struct vm_freelist
    vm_phys_free_queues[MAXMEMDOM][VM_NFREELIST][VM_NFREEPOOL][VM_NFREEORDER];
```

可以看到freelist 包含多层 第一层用于numa node的对应内存段的绑定，VM_NFREELIST端表示对应的内存块区间详见vm_phys_init，VM_NFREEPOOL对应的分配池。一般有两种分配池direct 和 default。UMA内存和page table 对应的page 尤direct 分配，default 用于常规分配页。 最后VM_NFREEORDER 表示对应的级数，分别为2页，4页，8页。

其中vm_freelist 是一个链表，每个链表的页数由VM_NFREEORDER 指定。并且与2的次方对齐。


# vm_phys_alloc_pages(int pool, int order)

vm_phys_alloc_pages 分配多个物理页。其中pool 指定对应的内存池（direct 或default 在amd64下）。order 指定页数为 2 << order 个页。

```
static vm_page_t
vm_phys_alloc_domain_pages(int domain, int flind, int pool, int order)
{	
	struct vm_freelist *fl;
	struct vm_freelist *alt;
	int oind, pind;
	vm_page_t m;

	mtx_assert(&vm_page_queue_free_mtx, MA_OWNED);
	fl = &vm_phys_free_queues[domain][flind][pool][0];
	for (oind = order; oind < VM_NFREEORDER; oind++) {
		m = TAILQ_FIRST(&fl[oind].pl);
		if (m != NULL) {
			vm_freelist_rem(fl, m, oind);
			vm_phys_split_pages(m, oind, fl, order);
			return (m);
		}
	}

	/*
	 * The given pool was empty.  Find the largest
	 * contiguous, power-of-two-sized set of pages in any
	 * pool.  Transfer these pages to the given pool, and
	 * use them to satisfy the allocation.
	 */
	for (oind = VM_NFREEORDER - 1; oind >= order; oind--) {
		for (pind = 0; pind < VM_NFREEPOOL; pind++) {
			alt = &vm_phys_free_queues[domain][flind][pind][0];
			m = TAILQ_FIRST(&alt[oind].pl);
			if (m != NULL) {
				vm_freelist_rem(alt, m, oind);
				vm_phys_set_pool(pool, m, oind);
				vm_phys_split_pages(m, oind, fl, order);
				return (m);
			}
		}
	}
	return (NULL);
}

// 其中pool 指定对应的内存池（direct 或default 在amd64下）。order 指定页数为 2 << order 个页。
vm_page_t
vm_phys_alloc_pages(int pool, int order)
{
	vm_page_t m;
	int domain, flind;
	struct vm_domain_iterator vi;

	KASSERT(pool < VM_NFREEPOOL,
	    ("vm_phys_alloc_pages: pool %d is out of range", pool));
	KASSERT(order < VM_NFREEORDER,
	    ("vm_phys_alloc_pages: order %d is out of range", order));

	vm_policy_iterator_init(&vi);

	// 选择对应的domain 如果domain 内存不足则
	while ((vm_domain_iterator_run(&vi, &domain)) == 0) {
		for (flind = 0; flind < vm_nfreelists; flind++) {
			m = vm_phys_alloc_domain_pages(domain, flind, pool,
			    order);
			if (m != NULL)
				return (m);
		}
	}

	vm_policy_iterator_finish(&vi);
	return (NULL);
}
```
