# 为什么要superpage

随着科技的日新月异，内存大小也在飞速的增长。而TLB的增长的速度却赶不上内存大小的增长速度。这就使得TLB的命中率严重下滑，为了避免这种情况，我们有了superpage。 

# superpage 操作
## Promotion

将512个连续的4K物理页面，聚合成一个2M的superpage （带有标记 PG_PS）。
合并有两个条件：
 - 所有物理页必须连续
 - 所有页必须包含相同的标志位

## Demotion

在内存紧缺时可以将一个2M的superpage，分割成512个连续的页。

## 什么时候需要合并物理页

在创建映射关系时，如果一个页的wire_count == 512表示有512个pte指向了同一个虚页面，或者是内核页面。且启用了大页，并且页面集群足够则可以进入promote。
	
