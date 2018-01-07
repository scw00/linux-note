代码
=====
```
pte_t *lookup_address_in_pgd(pgd_t *pgd, unsigned long address,
			     unsigned int *level)
{
	p4d_t *p4d;
	pud_t *pud;
	pmd_t *pmd;

	*level = PG_LEVEL_NONE;

	if (pgd_none(*pgd))
		return NULL;

	p4d = p4d_offset(pgd, address);
	if (p4d_none(*p4d))
		return NULL;

	*level = PG_LEVEL_512G;
	if (p4d_large(*p4d) || !p4d_present(*p4d))
		return (pte_t *)p4d;

	pud = pud_offset(p4d, address);
	if (pud_none(*pud))
		return NULL;

	*level = PG_LEVEL_1G;
	if (pud_large(*pud) || !pud_present(*pud))
		return (pte_t *)pud;

	pmd = pmd_offset(pud, address);
	if (pmd_none(*pmd))
		return NULL;

	*level = PG_LEVEL_2M;
	if (pmd_large(*pmd) || !pmd_present(*pmd))
		return (pte_t *)pmd;

	*level = PG_LEVEL_4K;

	return pte_offset_kernel(pmd, address);
}
```

可以看到在，映射对应地址时linux 更为倾向于先映射huge page（2M），如果没有发现则切换为4k的页继续匹配。
