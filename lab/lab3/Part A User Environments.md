# Part A: User Environments
这一部分是解决第一个进程的初始化以及进程上下文切换，总的来说需要思考的东西比较多，需要好好地捋清当前需要做什么，以及页表/目录的问题，也需要阅读参考资料（xv6等的）

总体上来说这部分实验分为三个部分：
1. 对全局变量的初始化
2. 创建初始化第一个进程
3. 进程切换

我们分别来看一下

env_init
---
```
// Mark all environments in 'envs' as free, set their env_ids to 0,
// and insert them into the env_free_list.
// Make sure the environments are in the free list in the same order
// they are in the envs array (i.e., so that the first call to
// env_alloc() returns envs[0]).
//
void
env_init(void)
{
	// Set up envs array
	// LAB 3: Your code here.
	struct Env *env=envs, *envs_end=envs+NENV;
	for(;env<envs_end;env++){
		env->env_id=0;
		env->env_link=env+1;
	}

	// set next of final env to NULL
	envs[NENV-1].env_link=NULL;

	// set golbal free start
	env_free_list=envs;

	// Per-CPU part of the initialization
	env_init_percpu();
}
```
相当于初始化进程表，没啥好说的

env_create
---
```
//
// Allocates a new env with env_alloc, loads the named elf
// binary into it with load_icode, and sets its env_type.
// This function is ONLY called during kernel initialization,
// before running the first user-mode environment.
// The new env's parent ID is set to 0.
//
void
env_create(uint8_t *binary, enum EnvType type)
{
	// LAB 3: Your code here.
	int errno;
	struct Env* e;

	// allocate a new env
	if((errno=env_alloc(&e,0))<0)
		panic("panic %e",errno);
	
	// set env type
	e->env_type=type;
	
	// load initial code to 
	// user space of env
	load_icode(e,binary);
}
```
这里可以分为两部分，首先是分配新进程，而后是加载代码到其地址空间，而分配新进程的部分与xv6类似，而且这里是所有进程共用一个内核栈的，也省去了着一些的麻烦，因此就不再分析了，只贴出其中关于设置内核页映射的部分：
```
//
// Initialize the kernel virtual memory layout for environment e.
// Allocate a page directory, set e->env_pgdir accordingly,
// and initialize the kernel portion of the new environment's address space.
// Do NOT (yet) map anything into the user portion
// of the environment's virtual address space.
//
// Returns 0 on success, < 0 on error.  Errors include:
//	-E_NO_MEM if page directory or table could not be allocated.
//
static int
env_setup_vm(struct Env *e)
{
	int i;
	struct PageInfo *p = NULL;

	// Allocate a page for the page directory
	if (!(p = page_alloc(ALLOC_ZERO)))
		return -E_NO_MEM;

	// Now, set e->env_pgdir and initialize the page directory.
	//
	// Hint:
	//    - The VA space of all envs is identical above UTOP
	//	(except at UVPT, which we've set below).
	//	See inc/memlayout.h for permissions and layout.
	//	Can you use kern_pgdir as a template?  Hint: Yes.
	//	(Make sure you got the permissions right in Lab 2.)
	//    - The initial VA below UTOP is empty.
	//    - You do not need to make any more calls to page_alloc.
	//    - Note: In general, pp_ref is not maintained for
	//	physical pages mapped only above UTOP, but env_pgdir
	//	is an exception -- you need to increment env_pgdir's
	//	pp_ref for env_free to work correctly.
	//    - The functions in kern/pmap.h are handy.

	// LAB 3: Your code here.
	p->pp_ref++;
	e->env_pgdir=KADDR(page2pa(p));

	if(setupkvm(e->env_pgdir)<0)
		return -E_NO_MEM;


	// UVPT maps the env's own page table read-only.
	// Permissions: kernel R, user R
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

	return 0;
}
```
其中setupkvm与原先初始化内核的页表设置部分相同：
```
int setupkvm(pde_t* pgdir){
	//////////////////////////////////////////////////////////////////////
	// Map 'pages' read-only by the user at linear address UPAGES
	// Permissions:
	//    - the new image at UPAGES -- kernel R, user R
	//      (ie. perm = PTE_U | PTE_P)
	//    - pages itself -- kernel RW, user NONE
	// Your code goes here:

	//define region size and initial pa
	uintptr_t UPAGES_size=ROUNDUP(npages*sizeof(struct PageInfo),PGSIZE),
			pages_pa=PADDR(pages);
	if(boot_map_region(pgdir,UPAGES,UPAGES_size,pages_pa,PTE_U)<0)
		return -E_NO_MEM;
		
	//////////////////////////////////////////////////////////////////////
	// Map the 'envs' array read-only by the user at linear address UENVS
	// (ie. perm = PTE_U | PTE_P).
	// Permissions:
	//    - the new image at UENVS  -- kernel R, user R
	//    - envs itself -- kernel RW, user NONE
	// LAB 3: Your code here.
	size_t UENVS_size=ROUNDUP(NENV*sizeof(struct Env),PGSIZE);
	physaddr_t envs_pa=PADDR(envs);
	if(boot_map_region(pgdir,UENVS,UENVS_size,envs_pa,PTE_U)<0)
		return -E_NO_MEM;

	//////////////////////////////////////////////////////////////////////
	// Use the physical memory that 'bootstack' refers to as the kernel
	// stack.  The kernel stack grows down from virtual address KSTACKTOP.
	// We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
	// to be the kernel stack, but break this into two pieces:
	//     * [KSTACKTOP-KSTKSIZE, KSTACKTOP) -- backed by physical memory
	//     * [KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE) -- not backed; so if
	//       the kernel overflows its stack, it will fault rather than
	//       overwrite memory.  Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	// Your code goes here:

	// define initial va and pa
	uintptr_t KSTACKBASE=KSTACKTOP-KSTKSIZE,bootstack_pa=PADDR(bootstack);
	if(boot_map_region(pgdir,KSTACKBASE,KSTKSIZE,bootstack_pa,PTE_W)<0)
		return -E_NO_MEM;

	// set page directory entry permission
	for(uint32_t n=0;n<KSTKSIZE;n+=PTSIZE)
		pgdir[PDX(KERNBASE+n)]|=PTE_W;

	//////////////////////////////////////////////////////////////////////
	// Map all of physical memory at KERNBASE.
	// Ie.  the VA range [KERNBASE, 2^32) should map to
	//      the PA range [0, 2^32 - KERNBASE)
	// We might not have 2^32 - KERNBASE bytes of physical memory, but
	// we just set up the mapping anyway.
	// Permissions: kernel RW, user NONE
	// Your code goes here:

#ifdef PSE_SUPPORT
	// define region size
	uintptr_t KERNSIZE=ROUNDUP((~KERNBASE)+1,LPGSIZE);
	boot_map_large_region(pgdir,KERNBASE,KERNSIZE,0x0,PTE_W);
#else
	// define region size
	uintptr_t KERNSIZE=(~KERNBASE)+1;
	if(boot_map_region(pgdir,KERNBASE,KERNSIZE,0x0,PTE_W)<0)
		return -E_NO_MEM;

	// set page directory entry permission
	for(n=0;n<KERNSIZE;n+=PTSIZE)
		pgdir[PDX(KERNBASE+n)]|=PTE_W;
#endif
	
	return 0;
}
```

而关于load_icode:
```
static void
load_icode(struct Env *e, uint8_t *binary)
{
	// LAB 3: Your code here.
	struct Elf* elf=(struct Elf*)binary;
	struct Proghdr *ph,*ph_start=(struct Proghdr*)(binary+elf->e_phoff),
		*ph_end=ph_start+elf->e_phnum;
	
	// The following loads each lodable program segment
	// First allocate pages for segments under e->env_pgdir
	// using region_alloc,then we have to switch to e->env_pgdir
	// to perform memmove because, the pages region_alloc has
	// just allocated are recorded on e->env_pgdir, not necessary
	// in kernerl space, which means, we cannot ensure that it can 
	// be accessed via kernerl pgdir(actually this almost never holds)
	// So it's necessary to first switch to e->env_pgdir to perfrom 
	// memmove(we can still access binary from there as binary resides in
	// kernel space) then switch back.

	// allocate physical pages correspoding to
	// program's memory size(rather than file size
	// as file size is smaller than memory size due
	// to segments like .bss)
	for(ph=ph_start;ph<ph_end;ph++)
		if(ph->p_type==ELF_PROG_LOAD)
			region_alloc(e,(void*)(ph->p_va),ph->p_memsz);

	// switch to env pgdir to access virtual address of phs
	lcr3(PADDR(e->env_pgdir));

	// copy program segmenta under env pgdir
	for(ph=ph_start;ph<ph_end;ph++)
		if(ph->p_type==ELF_PROG_LOAD){
			memmove((void*)(ph->p_va),(void*)(binary+ph->p_offset),ph->p_filesz);

			// set to 0 where memory size is larger than file size
			// due to segments like .bss
			memset((void*)(ph->p_va+ph->p_filesz),0,ph->p_memsz-ph->p_filesz);
		}

	// switch back to global pgdir
	lcr3(PADDR(kern_pgdir));

	// set eip of trapframe to the entry point
	e->env_tf.tf_eip=elf->e_entry;

	// Now map one page for the program's initial stack
	// at virtual address USTACKTOP - PGSIZE.

	// LAB 3: Your code here.
	region_alloc(e,(void*)(USTACKTOP-PGSIZE),PGSIZE);
}
```
只贴了自己实现的部分出来。这里比较重要的有两点：
1. 要注意初始化如.bss的段。这里我也是尝试了许多地方才找到怎么初始化的办法的，看一下被加载的段信息：
   	```
	filesize=0x4160
	flags=0x6
	memsize=0x4160
	offset=0x1000
	pa=0x200000
	type=0x1
	va=0x200000
   	```
	```
	filesize=0x10e0
	flags=0x5
	memsize=0x10e0
	offset=0x6020
	pa=0x800020
	type=0x1
	va=0x800020
	```
	```
	filesize=0x2c
	flags=0x6
	memsize=0x30
	offset=0x8000
	pa=0x802000
	type=0x1
	va=0x802000
	```
	这三个段有一个段的filesize<memsize，也就意味着这是数据段，而memsize比filesize大的那部分，就是bss段，因此初始化的时候要按照memsize分配空间，按照filesize拷贝代码，剩余的memsize-filesize要置0
2. 要切换页表才能进行源代码的拷贝，同时还要记得设置eip到程序的入口点（trapframe的其他成员都在分配进程的时候就设置好了）ps：这里切换页表部分能检测一下之前的页面映射是否有问题，这里花了我将近2、3个小时调试

以及region_alloc:
```
//
// Allocate len bytes of physical memory for environment env,
// and map it at virtual address va in the environment's address space.
// Does not zero or otherwise initialize the mapped pages in any way.
// Pages should be writable by user and kernel.
// Panic if any allocation attempt fails.
//
static void
region_alloc(struct Env *e, void *va, size_t len)
{
	// LAB 3: Your code here.
	// (But only if you need it for load_icode.)
	//
	// Hint: It is easier to use region_alloc if the caller can pass
	//   'va' and 'len' values that are not page-aligned.
	//   You should round va down, and round (va + len) up.
	//   (Watch out for corner-cases!)

	// rounding 
	void* rva=ROUNDDOWN(va,PGSIZE),*reva=ROUNDUP(va+len,PGSIZE);

	// bound check
	if(((uint32_t)rva)>=UTOP||((uint32_t)reva)>UTOP||reva<rva)
		panic("Illegal va=0x%x len=0x%x\n",va,len);

	struct PageInfo* pp;
	pte_t* pte;
	for(void* va=rva;va<reva;va+=PGSIZE){
		pp=page_alloc(0);
		pp->pp_ref++;

		pte=pgdir_walk(e->env_pgdir,va,1);
		if(!pte)
			panic("Fail to allocate page\n");
		
		*pte=(page2pa(pp)|PTE_U|PTE_W|PTE_P);
	}
}
```

env_run
---
这一部分也比较简单，因为只有一个内核栈，所以省去了保存内核栈指针的麻烦，而我们这里也还没有调度器，所以一个env_pop_tf就完成了跳转
```
void
env_run(struct Env *e)
{
	// Step 1: If this is a context switch (a new environment is running):
	//	   1. Set the current environment (if any) back to
	//	      ENV_RUNNABLE if it is ENV_RUNNING (think about
	//	      what other states it can be in),
	//	   2. Set 'curenv' to the new environment,
	//	   3. Set its status to ENV_RUNNING,
	//	   4. Update its 'env_runs' counter,
	//	   5. Use lcr3() to switch to its address space.
	// Step 2: Use env_pop_tf() to restore the environment's
	//	   registers and drop into user mode in the
	//	   environment.

	// Hint: This function loads the new environment's state from
	//	e->env_tf.  Go back through the code you wrote above
	//	and make sure you have set the relevant parts of
	//	e->env_tf to sensible values.

	// LAB 3: Your code here.
	if(curenv){
		if(curenv->env_status==ENV_RUNNING)
			curenv->env_status=ENV_RUNNABLE;
	}
	
	curenv=e;
	e->env_status=ENV_RUNNING;
	e->env_runs=0;
	lcr3(PADDR(e->env_pgdir));
	env_pop_tf(&e->env_tf);
	
	panic("env_run not yet implemented");
}
```

运行检测：
```
(gdb) x/16i $eip
=> 0xf010451e <env_pop_tf+21>:	popa   
   0xf010451f <env_pop_tf+22>:	pop    %es
   0xf0104520 <env_pop_tf+23>:	pop    %ds
   0xf0104521 <env_pop_tf+24>:	add    $0x8,%esp
   0xf0104524 <env_pop_tf+27>:	iret   
   0xf0104525 <env_pop_tf+28>:	lea    -0x87dcc(%ebx),%eax
   0xf010452b <env_pop_tf+34>:	push   %eax
   0xf010452c <env_pop_tf+35>:	push   $0x1f5
   0xf0104531 <env_pop_tf+40>:	lea    -0x87e22(%ebx),%eax
   0xf0104537 <env_pop_tf+46>:	push   %eax
   0xf0104538 <env_pop_tf+47>:	call   0xf01000b1 <_panic>
   0xf010453d <env_run>:	push   %ebp
   0xf010453e <env_run+1>:	mov    %esp,%ebp
   0xf0104540 <env_run+3>:	push   %ebx
   0xf0104541 <env_run+4>:	sub    $0x4,%esp
   0xf0104544 <env_run+7>:	call   0xf0100167 <__x86.get_pc_thunk.bx>
(gdb) si 1
=> 0xf010451f <env_pop_tf+22>:	pop    %es
0xf010451f in env_pop_tf (
    tf=<error reading variable: Unknown argument list address for `tf'.>)
    at kern/env.c:493
493		asm volatile(
(gdb) x/16i $eip
=> 0xf010451f <env_pop_tf+22>:	pop    %es
   0xf0104520 <env_pop_tf+23>:	pop    %ds
   0xf0104521 <env_pop_tf+24>:	add    $0x8,%esp
   0xf0104524 <env_pop_tf+27>:	iret   
   0xf0104525 <env_pop_tf+28>:	lea    -0x87dcc(%ebx),%eax
   0xf010452b <env_pop_tf+34>:	push   %eax
   0xf010452c <env_pop_tf+35>:	push   $0x1f5
   0xf0104531 <env_pop_tf+40>:	lea    -0x87e22(%ebx),%eax
   0xf0104537 <env_pop_tf+46>:	push   %eax
   0xf0104538 <env_pop_tf+47>:	call   0xf01000b1 <_panic>
   0xf010453d <env_run>:	push   %ebp
   0xf010453e <env_run+1>:	mov    %esp,%ebp
   0xf0104540 <env_run+3>:	push   %ebx
   0xf0104541 <env_run+4>:	sub    $0x4,%esp
   0xf0104544 <env_run+7>:	call   0xf0100167 <__x86.get_pc_thunk.bx>
   0xf0104549 <env_run+12>:	add    $0x8aad7,%ebx
(gdb) si 1
=> 0xf0104520 <env_pop_tf+23>:	pop    %ds
0xf0104520	493		asm volatile(
(gdb) si 1
=> 0xf0104521 <env_pop_tf+24>:	add    $0x8,%esp
0xf0104521	493		asm volatile(
(gdb) si 1
=> 0xf0104524 <env_pop_tf+27>:	iret   
0xf0104524	493		asm volatile(
(gdb) si 1
=> 0x800020:	cmp    $0xeebfe000,%esp
```
可以看到，iret结束后，虚拟地址跳转到0x800020，准备执行用户代码的第一句。

遵循lab上的指示，在int $0x30指令处设下断点，持续运行检测代码正确：
```
(gdb) b *0x800b44
Breakpoint 3 at 0x800b44
(gdb) c
Continuing.
=> 0x800b44:	int    $0x30

Breakpoint 3, 0x00800b44 in ?? ()
(gdb) 
```
运行良好