# Part2-3 Virtual Memory&Kernel Address Space
### Exercise 3
这一部分伊始先介绍了qemu的调试功能，包括MIT patch version提供的查看页表功能以及qemu自带的直接查看物理地址功能（因为可以看到gdb只能通过虚拟地址访问内存，这块儿可能不太方便）。

首先问题是，要怎么从JOS进入qemu monitor，因为之前我们都是通过gdb发送指令，qemu这边一直是陷在JOS里面的，指引上给出了方法，就是Ctrl-a组合然后再按一下c，进入qemu:
```
K> (qemu) 
```
首先试一下qemu对应gdb x/Nx的功能xp/Nx吧，可以看出多了个p应该是physical:
```
(qemu) xp/8x $esp
fffffffff0112e5c: 0x00000000 0x00000000 0x00000000 0x00000000
fffffffff0112e6c: 0x00000000 0x00000000 0x00000000 0x00000000
```
可以看出qemu print出来是64位地址，然而是用符号位扩展的，前面全是f，那我们强行offset一下：
```
(qemu) xp/8x $esp-0xfffffffff0000000
0000000000112e5c: 0xf010017c 0xf0114308 0xf01162a0 0xf0112e78
0000000000112e6c: 0xf010052b 0xf0102dd0 0xf0112e94 0xf0112e88
```
就有了我们要的物理地址。

接下来看一下页表:
```
(qemu) info pg
VPN range     Entry         Flags        Physical page
[00000-003ff]  PDE[000]     ----A----P
  [00000-00000]  PTE[000]     --------WP 00000
  [00001-0009f]  PTE[001-09f] ---DA---WP 00001-0009f
  [000a0-000b7]  PTE[0a0-0b7] --------WP 000a0-000b7
  [000b8-000b8]  PTE[0b8]     ---DA---WP 000b8
  [000b9-000ff]  PTE[0b9-0ff] --------WP 000b9-000ff
  [00100-00103]  PTE[100-103] ----A---WP 00100-00103
  [00104-00111]  PTE[104-111] --------WP 00104-00111
  [00112-00112]  PTE[112]     ---DA---WP 00112
  [00113-00115]  PTE[113-115] --------WP 00113-00115
  [00116-003ff]  PTE[116-3ff] ---DA---WP 00116-003ff
[f0000-f03ff]  PDE[3c0]     ----A---WP
  [f0000-f0000]  PTE[000]     --------WP 00000
  [f0001-f009f]  PTE[001-09f] ---DA---WP 00001-0009f
  [f00a0-f00b7]  PTE[0a0-0b7] --------WP 000a0-000b7
  [f00b8-f00b8]  PTE[0b8]     ---DA---WP 000b8
  [f00b9-f00ff]  PTE[0b9-0ff] --------WP 000b9-000ff
  [f0100-f0103]  PTE[100-103] ----A---WP 00100-00103
  [f0104-f0111]  PTE[104-111] --------WP 00104-00111
  [f0112-f0112]  PTE[112]     ---DA---WP 00112
  [f0113-f0115]  PTE[113-115] --------WP 00113-00115
  [f0116-f03ff]  PTE[116-3ff] ---DA---WP 00116-003ff
```
可以看到，从左到右分别是虚拟页号范围，PDE/PTE index范围，flag，以及对应的物理页号范围，同样flag的合并到一个范围显示，非常贴心。相比之下qemu自带的mem就非常简陋了：
```
(qemu) info mem
0000000000000000-0000000000400000 0000000000400000 -r-
00000000f0000000-00000000f0400000 0000000000400000 -rw
```
### Question
- Assuming that the following JOS kernel code is correct, what type should variable x have, uintptr_t or physaddr_t?
	mystery_t x;
	char* value = return_a_pointer();
	*value = 10;
	x = (mystery_t) value;

这里比较简单，x的类型应该是uintptr_t，函数返回的只有virtual address，真正访问物理地址从来都是拿到virtual address后由MMU代为转化的。这里把10赋给value指向的字节时，也是通过MMU先把value这个va转化成physical address再访问的。这一切在开启了cr0的PE位并设置好cr3寄存器（page directory的base address）后对我们来说就是transparent的了。

---
还是首先明确一下这里我们已经有什么，要做什么。

通过Part 1，我们已经实现了对物理页的track，分配，回收。那么下一步就是要初始化页表映射了。
在之前我们用的是在kern/entrydir.c中hard code的Page Directory和Page Tables，大致如下：
```
pde_t entry_pgdir[NPDENTRIES] = {
	// Map VA's [0, 4MB) to PA's [0, 4MB)
	[0]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
	[KERNBASE>>PDXSHIFT]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};
```
```
pte_t entry_pgtable[NPTENTRIES] = {
	0x000000 | PTE_P | PTE_W,
	0x001000 | PTE_P | PTE_W,
	0x002000 | PTE_P | PTE_W,
	0x003000 | PTE_P | PTE_W,
  ...
  0x3fe000 | PTE_P | PTE_W,
	0x3ff000 | PTE_P | PTE_W,
};
```
可以看到，设置了pgdir的第0项和第KERNBASE>>PDXSHIFT项，恰好对应0x0开始的一个PTSIZE和0xf0000000开始的一个PTSIZE，而且这两个Page Directory Entry是指向的同一个Page Table，只不过权限不一样

那么接下来我们就要从头开始建立页表映射了，想想还有点小激（tou）动(tong)呢.
我们分成两部分来实现，首先实现一些虚拟内存管理的功能函数，然后用这些功能函数来初始化页表/页目录，最后用新的切换到这个新的页表/页目录下。（在实现的过程中要意识到，我们现在仍然在用之前的hard code映射，直到我们实现完并且切换，才会采用我们现在实现的机制，不要搞混虚拟内存与物理内存地址的映射关系）

---
- pgdir_walk。返回一个va对应的页表项指针（注意是页表项指针，不是页表指针，函数上面的注释有点小问题）

这里首先要明确的是，既然要返回页表项指针，那得先有页表才行，因此没有页表会先创建，所以这个函数是有side effect的。
```
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
	int pdx=PDX(va),ptx=PTX(va);
	
	// check Page Table page present
	if (pgdir[pdx] & PTE_P){
		// when present retrieve pa of page table
		// and add an appropiate offset i.e. ptx
		return ((pte_t*)(KADDR(PTE_ADDR(pgdir[pdx]))))+ptx;
	
	// when required to create
	}else if(create){
		// non-present and required to create
		// allocate a page for page table
		struct PageInfo* ptp_info=page_alloc(ALLOC_ZERO);// info of page for page table
		
		// check successful page allocation
		if(ptp_info){
			// update reference count
			ptp_info->pp_ref++;

			// set page directory entry
			physaddr_t ptp_pa=page2pa(ptp_info);
			pgdir[pdx]=(pde_t)(ptp_pa|PTE_P|PTE_W|PTE_U);

			// return the corresponding page table entry
			// (Note that's not page table va but page table
			// entry va)
			return ((pte_t*)KADDR(ptp_pa))+ptx;
		}
	}
	
	return NULL;
}
```
首先是看一下是否present，present的话直接抽取、转换就可以了。要注意的是这里从pgdir中获取到page table base address后要先转化为虚拟地址再返回，起初我就是漏了KADDR宏，返回了物理地址导致内存错误。

non-present又要求create的话，我们需要做的就是三件事：
1. 分配物理页给页表
2. 1成功的话，更新下列信息：
  1. 物理页对应的Pageinfo信息
  2. pgdir中对应于这个page table的entry
3. 根据分配到的页表的起始地址计算出va对应的pte的**虚拟**地址并返回

这里要注意pde的权限，要有写入权，要给用户态访问权（即便我们不知道这个pde对应的page table用户态是否有权访问也无妨，等到访问pte的时候仍然会有权限检查），以及最后的，返回的不是页表其实地址而是页表项地址（而且是虚拟地址）
- boot_map_region，将连续的虚拟页映射到连续的物理页，方便后面连续区域的映射

有了之前的pgdir_walk，这个实现起来就非常方便了。
```
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	int n;
	pte_t* va_pte;
	for(n=0;n<size;n+=PGSIZE){
		// here pgdir_walk will automatically
		// allocate pages for page tables
		va_pte=pgdir_walk(pgdir,(void*)(va+n),1);		

		// check page allocation
		if (!va_pte)
			panic("Fail to allocate physical memory page.\n");
		
		// update page table entry
		*va_pte=(pte_t)((pa+n)|perm|PTE_P);	
	}
}
```
- page_insert，把一个虚拟地址va开始的page映射到一个Pageinfo对应的物理页上

这里，要考虑的一个corner case是，va原本就已经映射到一个pp上了，对引用的处理就要小心。
```
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
	pte_t* va_pte=pgdir_walk(pgdir,va,1);
	if (!va_pte)
		return -E_NO_MEM;
	
	// check whether va is not mapped to pp;
	// non-corner case
	if(page2pa(pp)!=PTE_ADDR(*va_pte)){
		page_remove(pgdir,va);
		tlb_invalidate(pgdir,va);
		pp->pp_ref++;
	}
	
	// set up page table entry
	*va_pte=(page2pa(pp)|perm|PTE_P);
	
	return 0;
}
```
这里没有想到注释里说的elegant way来用单一code path来解决corner case的方法，还是用了if...

当遇到corner case的时候仅仅设置页表项，其他的都不用干
- page_lookup，对pgdir_walk进行了封装，但查询得到的pte*通过输入参数修改，返回值变为物理页对应的PageInfo地址
```
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
	// look up via pgdir_work
	pte_t* va_pte=pgdir_walk(pgdir,va,0);
	if(va_pte&&((*va_pte)&PTE_P)){
		//when required to stor pte_t*
		if(pte_store)
			*pte_store=va_pte;

		// return corresponding Paginfo*
		return pa2page(PTE_ADDR(*va_pte));
	}
	
	return NULL;
}
```
- page_remove，（如果存在的话）取消va的物理页映射

```
void
page_remove(pde_t *pgdir, void *va)
{
	pte_t* va_pte;
	struct PageInfo*
		vap_info=page_lookup(pgdir,va,&va_pte);

	// perform removing only if there is a
	// physical page mapped to va
	if(vap_info){
		// decrease reference count,
		// clear page table entry,
		// invalidate TLB
		page_decref(vap_info);
		*va_pte=0;
		tlb_invalidate(pgdir,va);
	}
}
```
至此，所有功能函数算是定义完了，下面开始初始化页表/页目录

---
要先想清楚的是，有哪些虚拟地址是需要初始化页表/页目录项的？答案也很简单，已经用了的肯定要初始化，其他的有必要的也要初始化。这个“其他的”在这里指的就是UPAGES开始处的地址了。因为pages是在内核空间中的，用户态无法访问，因此我们在用户态区域UPAGES开始处做一个pages的镜像，用户态通过虚拟地址访问，但其实是映射到了pages所在的物理页，这样，用户态通过UPAGES访问pages所在的物理页，内核态通过pages访问pages所在的物理页，而前者是只读的权限，后者具有读写权限.
```
//define region size and initial pa
uintptr_t UPAGES_size=ROUNDUP(npages*sizeof(struct PageInfo),PGSIZE),
    pages_pa=PADDR(pages);

boot_map_region(kern_pgdir,UPAGES,UPAGES_size,pages_pa,PTE_U);
```
接下来就是“已经用了的”，这里我们只初始化内核需要的，包括两部分，一个是KERNBASE(0xf0000000)往上的，内核的代码数据等等的，一个是KERNBASE往下延申的内核栈，因此初始化如下：
```
// define initial va and pa
uintptr_t KSTACKBASE=KSTACKTOP-KSTKSIZE,bootstack_pa=PADDR(bootstack);
boot_map_region(kern_pgdir,KSTACKBASE,KSTKSIZE,bootstack_pa,PTE_W);

// set page directory entry permission
for(n=0;n<KSTKSIZE;n+=PTSIZE)
  kern_pgdir[PDX(KERNBASE+n)]|=PTE_W;
```
这是初始化内核栈的，值得注意的是，这里多了一处初始化pde的语句，因为默认的pde权限是不可写，而这里我们要给内核写权限，因此需要在初始化页表外额外处理一下
```
// define region size
uintptr_t KERNSIZE=(~KERNBASE)+1;
boot_map_region(kern_pgdir,KERNBASE,KERNSIZE,0x0,PTE_W);

// set page directory entry permission
for(n=0;n<KERNSIZE;n+=PTSIZE)
  kern_pgdir[PDX(KERNBASE+n)]|=PTE_W;
```
初始化KERNBASE以上同理

至此，内核物理地址初始化完成

```
Physical memory: 131072K available, base = 640K, extended = 130432K
check_page_free_list() succeeded!
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_free_list() succeeded!
check_page_installed_pgdir() succeeded!
Welcome to the JOS kernel monitor!
```
### Question
- What entries (rows) in the page directory have been filled in at this point? What addresses do they map and where do they point? In other words, fill out this table as much as possible:

| Entry | Base Virtual Address |                                             Points to (logically)                                             |
| :---: | :------------------: | :-----------------------------------------------------------------------------------------------------------: |
| 1023  |      0xffc00000      |                        page table for 1024th(last) 4MB virtual memory of kernerl sapce                        |
| 1022  |      0xff800000      |                           page table for 1023th 4MB virtual memory of kernerl sapce                           |
|   .   |          .           |                                                       .                                                       |
|   .   |          .           |                                                       .                                                       |
|  961  |      0xf0400000      |                            page table for 2nd 4MB virtual memory of kernerl sapce                             |
|  960  |      0xf0000000      |                            page table for 1st 4MB virtual memory of kernerl sapce                             |
|  959  |      0xefc00000      | page table for the area including  kernel stacks and invalid memory chunks(buffer  in case of stack overflow) |
|  958  |      0xef800000      |                                             page directory itself                                             |
|  957  |      0xef400000      |          page table for read-only user environment pages(physical page allocation information) image          |
|  956  |      0xef000000      |                                                       ?                                                       |
|   .   |          .           |                                                       .                                                       |
|   .   |          .           |                                                       .                                                       |
|   2   |      0x00c00000      |                                                       ?                                                       |
|   1   |      0x00400000      |                                                       ?                                                       |
|   0   |      0x00000000      |                                                       ?                                                       |

- We have placed the kernel and user environment in the same address space. Why will user programs not be able to read or write the kernel's memory? What specific mechanisms protect the kernel memory?

用户程序不能访问内核内存区域主要是为了保证计算机正常运行，有必要访问内核的时候全部系统调用，由内核态代为执行。保护机制主要是页表/页目录中的标志位，MMU会检查标志位，非法访问被禁止
- What is the maximum amount of physical memory that this operating system can support? Why?

这里很明显就是4GB了（如果hole区域也算进来的话），这是由page directory和page table，page的大小共同决定的
- How much space overhead is there for managing memory, if we actually had the maximum amount of physical memory? How is this overhead broken down?

overhead可以分以下这些来考虑：
1. 物理页管理，Pageinfo一个8字节，满内存的话1M个页，overhead=8MB
2. 页目录开销，一个项4字节，不管满没满内存都需要1K个项，overhead=4KB
3. 页表开销，这个主要取决于你现在程序消耗了多少内存，而且是否是连续的内存地址消耗（不连续的话要多开页表），假如全部页表都被开出来，那就是1K个页表，每个页表4KB，overhead=4MB

因此最大的overhead是8MB+4KB+4MB=12292KB，可以broken down为上述三部分
- Revisit the page table setup in kern/entry.S and kern/entrypgdir.c. Immediately after we turn on paging, EIP is still a low number (a little over 1MB). At what point do we transition to running at an EIP above KERNBASE? What makes it possible for us to continue executing at a low EIP between when we enable paging and when we begin running at an EIP above KERNBASE? Why is this transition necessary?

第一个问题之前已经说过了，是在jmp $relocated之后EIP进入高地址，之所以EIP低地址也可以访问是因为hard code的映射把间隔0xf0000000的高低地址都映射到相同物理内存，因此是可以正常运行的。而这个转换是为了后面进入C语言代码做准备，到那时所有内存访问都是通过虚拟地址。