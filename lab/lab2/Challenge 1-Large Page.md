# Challenge 1: Large Page
> **Challenge! We consumed many physical pages to hold the page tables for the KERNBASE mapping. Do a more space-efficient job using the PTE_PS ("Page Size") bit in the page directory entries. This bit was not supported in the original 80386, but is supported on more recent x86 processors. You will therefore have to refer to Volume 3 of the current Intel manuals. Make sure you design the kernel to use this optimization only on processors that support it!**

因为没有具体的要求，所以这里就仅实现了
1. 一个可以分配4MB页表的large_page_alloc(int alloc_flags)
2. 内核空间的分配采用了4MB的large page
3. 根据变动修改了部分原先的函数

intel手册上说明，开启cr4的PSE可以enable 4MB的页，而且可以与原先的4KB页共存，具体根据page directory上的PS位来决定，而关于4MB页的使用，上面给出了这样一条建议:
> A typical example of mixing 4-KByte and 4-MByte pages is to place the operating system or executive’s kernel in a large page to reduce TLB misses and thus improve overall system performance. The processor maintains 4-MByte page entries and 4-KByte page entries in separate TLBs. So, placing often used code such as the kernel in a large page, frees up 4-KByte-page TLB entries for application programs and tasks.

这里我们就按照它的建议来

首先要能够分配连续的4MB页面，因此是一个寻找4MB页面的函数：
```
#ifdef PSE_SUPPORT
struct PageInfo* 
large_page_alloc(int alloc_flags){

	// Fail when no free page
	if(!page_free_list)
		return NULL;

	// start: start of pp w.r.t continuous pages to be allocated
	// current: track pp pointer
	// end: end of pages 
	// jmp: pp whose pp_link is "start"
 	struct PageInfo *start,*current,*end,*jmp;
	jmp=start=current=page_free_list;
	end=pages+npages;
	
	int cnt;
	// bound check
	while (current<end){
		cnt=0;
		// stop when at least one conditions met:
		// 1. array overflow(current>=end)
		// 2. pages to allocated is broken by an aleady occupied page(current->pp_link==NULL)
		// 3. page count reachs NPTENTRIES(1024)
		while(current<end&&current->pp_link&&cnt<NPTENTRIES){
			current++;
			cnt++;
		}

		// when while-loop is broken for adequate pages
		// deal with pointer update and return
		if (cnt==NPTENTRIES){

			// This implies the continuous pages we found
			// start just at first free page, so no front
			// pointer needs to be updated, just update 
			// the page_free_list
			if(jmp==page_free_list)
				page_free_list=(current-1)->pp_link;
			
			// This maybe the most case, we don't update
			// the page_free_list but the one previously 
			// points to our found area, and set it point
			// to next free page
			else
				jmp->pp_link=(current-1)->pp_link;
			
			// set "occupied" status for our found pages
			for(end=start;end<current;end++)
				end->pp_link=NULL;

			// return the "start"
			return start;

		// here it turns out that cnt<NPTENTRIES,
		// so we only perform bound check
		}else if(current<end){

			// pp just before "current" is the one
			// points to the new "start" found below
			jmp=current-1;

			// find new "start" by bypassing
			// all occupied pages with bound check
			while(!(current->pp_link)&&current<end)
				current++;

			// new "next", back to the outer while-loop
			// restart finding continuous 1024 empty pages
			start=current;
		}
	}

	return NULL;
}
#endif
```
这里的寻找看起来有点蠢，没有用buddy system，那个我在challenge 3中进行了实现，但并没有用在这里，主要有两个考虑：
1. 虽然buddy system里设置的最大连续page frame是1024也即4MB，但是这个数字是不太可靠的，如果只有512那可能就没法依赖了
2. 这个page 分配是用于初始化阶段的，大部分物理内存都是空白的，因此区别也不大，后续只应当使用malloc和free，因此也无碍

而对于物理页的分配，仅一处改用了这里的large page，即KERNBASE往上的空间：
```
#ifdef PSE_SUPPORT
	// define region size
	uintptr_t KERNSIZE=ROUNDUP((~KERNBASE)+1,LPGSIZE);
	boot_map_large_region(kern_pgdir,KERNBASE,KERNSIZE,0x0,PTE_W);
#else
	// define region size
	uintptr_t KERNSIZE=(~KERNBASE)+1;
	boot_map_region(kern_pgdir,KERNBASE,KERNSIZE,0x0,PTE_W);

	// set page directory entry permission
	for(n=0;n<KERNSIZE;n+=PTSIZE)
		kern_pgdir[PDX(KERNBASE+n)]|=PTE_W;
#endif
```
其中boot_map_large_region如下：
```
#ifdef PSE_SUPPORT
static void
boot_map_large_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	int n;
	for(n=0;n<size;n+=LPGSIZE)
		pgdir[PDX(va+n)]=(pde_t)((pa+n)|perm|PTE_P|PTE_PS);		
}
#endif
```
仅设置pde即可。最后就是开启PSE位了：
```
#ifdef PSE_SUPPORT
	// enable optional 4MB page
	lcr4(rcr4()|CR4_PSE);
#endif

	// load new pgdir to replace previous hard-code one
	lcr3(PADDR(kern_pgdir));
```
注意这里要先enbale PSE再load kern_pgdir到cr3寄存器，否则内核区域的地址转换会出致命问题（PDE其实是PTE，PT是空的）

在之前的代码中也有体现，使用宏来控制是否支持PSE：
```
#ifndef PSE_SUPPORT
#define PSE_SUPPORT
#endif
#ifdef PSE_SUPPORT
#define KPGSIZE (LPGSIZE)
#else
#define KPGSIZE (PGSIZE)
#endif
```
为了使4MB与4KB混用的物理页能通过check，需要魔改一下check的代码，首先是测试中va到pa的转换方法，也即是check_va2pa()：
```
static physaddr_t
check_va2pa(pde_t *pgdir, uintptr_t va)
{
	pte_t *p;

	pgdir = &pgdir[PDX(va)];
	if (!(*pgdir & PTE_P))
		return ~0;

#ifdef PSE_SUPPORT
	if(!(*pgdir&PTE_PS)){
		p = (pte_t*) KADDR(PTE_ADDR(*pgdir));
		if (!(p[PTX(va)] & PTE_P))
			return ~0;
		return PTE_ADDR(p[PTX(va)]);
	}else
		return PTE_ADDR(*pgdir);
#else
	p = (pte_t*) KADDR(PTE_ADDR(*pgdir));
	if (!(p[PTX(va)] & PTE_P))
		return ~0;
	return PTE_ADDR(p[PTX(va)]);
#endif
}
```
而后是初始化/测试代码中对内核页的访问粒度:
```
//set allocated status
size_t i_start=IOPHYSMEM/PGSIZE,i_end=ROUNDUP(PADDR(boot_alloc(0)),KPGSIZE)/PGSIZE;
for(i=i_start;i<i_end;i++){
    pages[i].pp_ref=1;
    pages[i].pp_link=NULL;
}
```
```
// check phys mem
for (i = 0; i < npages * PGSIZE; i += KPGSIZE)
    assert(check_va2pa(pgdir, KERNBASE + i) == i);
```
都由PGSIZE改为定义的KPGSIZE

一通改动后:
```
Physical memory: 131072K available, base = 640K, extended = 130432K
check_page_free_list() succeeded!
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_free_list() succeeded!
check_page_installed_pgdir() succeeded!
check_malloc_and_free() succeeded!
```

