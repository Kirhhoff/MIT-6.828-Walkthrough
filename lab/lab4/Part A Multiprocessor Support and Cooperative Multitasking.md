# Part A: Multiprocessor Support and Cooperative Multitasking

这部分的内容主要是对多处理器的支持以及基本的进程调度，比较麻烦的应该是与之前代码的适配

先分析一下执行流程吧。在启动时只有一个cpu激活，我们之前所做的一切都运行在这个cpu上，init的流程再看一下：
```
void
i386_init(void)
{
	// Initialize the console.
	// Can't call cprintf until after we do this!
	cons_init();

	cprintf("6828 decimal is %o octal!\n", 6828);

	// Lab 2 memory management initialization functions
	mem_init();
	
	msr_init();
	
	// Lab 3 user environment initialization functions
	env_init();
	trap_init();

	// Lab 4 multiprocessor initialization functions
	mp_init();
	lapic_init();

	// Lab 4 multitasking initialization functions
	pic_init();

	// Acquire the big kernel lock before waking up APs
	// Your code here:
	lock_kernel();

	// Starting non-boot CPUs
	boot_aps();

#if defined(TEST)
	// Don't touch -- used by grading script!
	ENV_CREATE(TEST, ENV_TYPE_USER);
#else
	// Touch all you want.
	ENV_CREATE(user_primes, ENV_TYPE_USER);
#endif // TEST*

	// Schedule and run the first user environment!
	sched_yield();
}
```
除了我们之前的内存、进程、中断设置，接下来就是多处理器设置以及启动它们了，这部分是由mp_init和boot_aps执行的。总的来说，mp_init主要检测cpu的存在以及初始化全局cpus数组，boot_aps启动它们，但是注意，每个cpu都要先初始化它的本地中断管理器也就是local apic，每个处理器都要执行一次。接着由boot cpu启动它们，将其启动地点设置为mpentry_start，而mpentry_start即是mpentry.S文件中的代码，而后跳转到mp_main中执行

> Exercise 1. Implement mmio_map_region in kern/pmap.c. To see how this is used, look at the beginning of lapic_init in kern/lapic.c. You'll have to do the next exercise, too, before the tests for mmio_map_region will run.

这一部分是实现IO映射。我们的目标是在内核空间中预留出来一部分虚拟内存，当访问这部分虚拟内存时实际上是在访问这部分IO端口。以lapic端口为例，起始于物理地址0xFE000000，大小为4KB，我们需要做的是分配一段虚拟地址va，当访问va时，实际在访问物理地址0xFE000000，这里可能会有一些疑惑，如果没有4GB内存怎么办呢？不用担心，即便只有3GB内存，当访问0xFE000000时也会导向到lapic设备，硬件已经进行了映射。
```
void *
mmio_map_region(physaddr_t pa, size_t size)
{
	// Where to start the next region.  Initially, this is the
	// beginning of the MMIO region.  Because this is static, its
	// value will be preserved between calls to mmio_map_region
	// (just like nextfree in boot_alloc).
	static uintptr_t base = MMIOBASE;

	// Reserve size bytes of virtual memory starting at base and
	// map physical pages [pa,pa+size) to virtual addresses
	// [base,base+size).  Since this is device memory and not
	// regular DRAM, you'll have to tell the CPU that it isn't
	// safe to cache access to this memory.  Luckily, the page
	// tables provide bits for this purpose; simply create the
	// mapping with PTE_PCD|PTE_PWT (cache-disable and
	// write-through) in addition to PTE_W.  (If you're interested
	// in more details on this, see section 10.5 of IA32 volume
	// 3A.)
	//
	// Be sure to round size up to a multiple of PGSIZE and to
	// handle if this reservation would overflow MMIOLIM (it's
	// okay to simply panic if this happens).
	//
	// Hint: The staff solution uses boot_map_region.
	//
	// Your code here:
	size=ROUNDUP(size,PGSIZE);
	pa=ROUNDDOWN(pa,PGSIZE);

	// bound check
	if(base+size>=MMIOLIM)
		panic("Memory-mapped address exceeds MMIOLIM.");
	
	boot_map_region(kern_pgdir,base,size,pa,PTE_W|PTE_PCD|PTE_PWT);
	base+=size;
	
	// return address before increment
	return (void*)(base-size);
}
```
主要内容就是从MMIOBASE开始往上为IO分配映射，并检测不超过MMIOLIM。
但是这里还有其他需要做的，就是在setupkvm时，也就是分配一个新的pgdir时，共享映射这部分IO虚拟地址，这部分代码如下(setupkvm)：
```
	pgdir[PDX(MMIOBASE)]=kern_pgdir[PDX(MMIOBASE)];// map memory-mapped IO ports
```
完成了IO端口映射，接下来我们把分析一下其他处理器启动时的行为
> Exercise 2. Read boot_aps() and mp_main() in kern/init.c, and the assembly code in kern/mpentry.S. Make sure you understand the control flow transfer during the bootstrap of APs. Then modify your implementation of page_init() in kern/pmap.c to avoid adding the page at MPENTRY_PADDR to the free list, so that we can safely copy and run AP bootstrap code at that physical address. Your code should pass the updated check_page_free_list() test (but might fail the updated check_kern_pgdir() test, which we will fix soon).

之前我们已经概括过boot_aps做了什么，而我们现在来看一下其中的同步细节：
```
	// Write entry code to unused memory at MPENTRY_PADDR
	code = KADDR(MPENTRY_PADDR);
	memmove(code, mpentry_start, mpentry_end - mpentry_start);

	// Boot each AP one at a time
	for (c = cpus; c < cpus + ncpu; c++) {
		if (c == cpus + cpunum())  // We've started already.
			continue;

		// Tell mpentry.S what stack to use 
		mpentry_kstack = percpu_kstacks[c - cpus] + KSTKSIZE;
		// Start the CPU at mpentry_start
		lapic_startap(c->cpu_id, PADDR(code));
		// Wait for the CPU to finish some basic setup in mp_main()
		while(c->cpu_status != CPU_STARTED)
			;
	}
```
注意这里lapic_startap给要启动的cpu发送了一个中断信号使其开始从mpentry.S代码开始执行，接着就开始“忙等待”直到这个cpu的状态被那个刚启动的cpu修改为STARTED才接着启动下一个cpu。而这里也可以看到，mpentry_kstack被初始化到了内核区域而不是按照内存分布图里说的KSTACK_TOP下面，这个没关系，我们下面会说明并解决这个问题。至此，我们需要把mpentry代码的物理页预留出来：
```
	pages[MPENTRY_PADDR/PGSIZE].pp_ref=1;
	pages[MPENTRY_PADDR/PGSIZE].pp_link=NULL;
	pages[MPENTRY_PADDR/PGSIZE-1].pp_link=&pages[MPENTRY_PADDR/PGSIZE+1];
```

Question
---
1. Compare kern/mpentry.S side by side with boot/boot.S. Bearing in mind that kern/mpentry.S is compiled and linked to run above KERNBASE just like everything else in the kernel, what is the purpose of macro MPBOOTPHYS? Why is it necessary in kern/mpentry.S but not in boot/boot.S? In other words, what could go wrong if it were omitted in kern/mpentry.S?
Hint: recall the differences between the link address and the load address that we have discussed in Lab 1.

这也很简单，与先前boot.S中一样。因为每个处理器都有自己单独的cr0,cr3寄存器，在cpu的这个阶段，分页还没有开启，地址访问还是物理内存，因此需要重定位，当然，这里的重定位地址可能与先前的值不太一样，但目的是相同的。

接下来就是为所有处理器映射内核栈了，截至目前，我们的esp还指在内核区域里，我们接下来在内核页中为KERNBASE下面的内存区域作映射，映射到percpu_kstacks的实际物理页上，这样，当我们访问KERNBASE下面的非guard部分的stack时，实际访问到的就是percpu_kstacks的实际物理内存部分了。虽然这并不会改变esp的指向，但是也很有用，我们下面再说，先把这个干完
> Exercise 3. Modify mem_init_mp() (in kern/pmap.c) to map per-CPU stacks starting at KSTACKTOP, as shown in inc/memlayout.h. The size of each stack is KSTKSIZE bytes plus KSTKGAP bytes of unmapped guard pages. Your code should pass the new check in check_kern_pgdir().
```
// Modify mappings in kern_pgdir to support SMP
//   - Map the per-CPU stacks in the region [KSTACKTOP-PTSIZE, KSTACKTOP)
//
static void
mem_init_mp(void)
{
	// Map per-CPU stacks starting at KSTACKTOP, for up to 'NCPU' CPUs.
	//
	// For CPU i, use the physical memory that 'percpu_kstacks[i]' refers
	// to as its kernel stack. CPU i's kernel stack grows down from virtual
	// address kstacktop_i = KSTACKTOP - i * (KSTKSIZE + KSTKGAP), and is
	// divided into two pieces, just like the single stack you set up in
	// mem_init:
	//     * [kstacktop_i - KSTKSIZE, kstacktop_i)
	//          -- backed by physical memory
	//     * [kstacktop_i - (KSTKSIZE + KSTKGAP), kstacktop_i - KSTKSIZE)
	//          -- not backed; so if the kernel overflows its stack,
	//             it will fault rather than overwrite another CPU's stack.
	//             Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	//
	// LAB 4: Your code here:
	int cpui,kstackbase,kstackbottom_pa,pagei;
	pte_t* pte;
	for(cpui=0;cpui<NCPU;cpui++){
		// va and pa with respect to 
		// bottom of i'th cpu kernel stack
		kstackbase=KSTACKTOP-(cpui+1)*(KSTKSIZE+KSTKGAP);
		kstackbottom_pa=PADDR((void*)percpu_kstacks[cpui]);

		// map stack content
		boot_map_region(kern_pgdir,kstackbase+KSTKGAP,KSTKSIZE,kstackbottom_pa,PTE_W);

		// map stack guard gaps
		// pte set to 0 works as guard area
		for(pagei=0;pagei<KSTKGAP;pagei+=PGSIZE){
			// check for pte existence
			if((pte=pgdir_walk(kern_pgdir,(void*)kstackbase+pagei,0))==0)
				panic("pgdir_walk for kernel stack gap fails.");
			
			// clear it as guard area
			*pte=0;
		}	
	}
}
````
其中比较重要的部分应该是将每个cpu的stack部分与gurad部分分开设置，前者予以正确的权限，后者直接也不分配物理内存而设为NULL，起到保护作用

注意这一部分的映射工作是在启动其他处理器之前就进行完了的，至于为什么在boot_aps中仍然使用percpu_kstacks作为esp值，我想应该是因为影响不大，因为那部分代码不会使用很多栈内存，不会溢出，而我们只要在这个cpu的TSS中把esp0设置到KERNBASE下面正确的区域，就可以保证每次进入内核态时都能加载正确的esp值从而使得guard部分起到保护作用。也即是，这部分代码只是临时的，没什么影响， 这样写还更加简洁一些。

另外还需要在setupkvm中每次设置新的pgdir时加上对内核栈的映射：
```
	pgdir[PDX(MMIOLIM)]=kern_pgdir[PDX(MMIOLIM)];// map kernel stacks for cpus 
```

做了所有这些前置部分，我们就开始启动cpu前的最后一步，设置每个CPU的TSS了

> Exercise 4. The code in trap_init_percpu() (kern/trap.c) initializes the TSS and TSS descriptor for the BSP. It worked in Lab 3, but is incorrect when running on other CPUs. Change the code so that it can work on all CPUs. (Note: your new code should not use the global ts variable any more.)
```
// Initialize and load the per-CPU TSS and IDT
void
trap_init_percpu(void)
{
	// The example code here sets up the Task State Segment (TSS) and
	// the TSS descriptor for CPU 0. But it is incorrect if we are
	// running on other CPUs because each CPU has its own kernel stack.
	// Fix the code so that it works for all CPUs.
	//
	// Hints:
	//   - The macro "thiscpu" always refers to the current CPU's
	//     struct CpuInfo;
	//   - The ID of the current CPU is given by cpunum() or
	//     thiscpu->cpu_id;
	//   - Use "thiscpu->cpu_ts" as the TSS for the current CPU,
	//     rather than the global "ts" variable;
	//   - Use gdt[(GD_TSS0 >> 3) + i] for CPU i's TSS descriptor;
	//   - You mapped the per-CPU kernel stacks in mem_init_mp()
	//   - Initialize cpu_ts.ts_iomb to prevent unauthorized environments
	//     from doing IO (0 is not the correct value!)
	//
	// ltr sets a 'busy' flag in the TSS selector, so if you
	// accidentally load the same TSS on more than one CPU, you'll
	// get a triple fault.  If you set up an individual CPU's TSS
	// wrong, you may not get a fault until you try to return from
	// user space on that CPU.
	//
	// LAB 4: Your code here:

	// Setup a TSS so that we get the right stack
	// when we trap to the kernel.
	thiscpu->cpu_ts.ts_esp0=KSTACKTOP-cpunum()*(KSTKSIZE+KSTKGAP);
	thiscpu->cpu_ts.ts_ss0=GD_KD;
	thiscpu->cpu_ts.ts_iomb=sizeof(struct Taskstate);

	// Initialize the TSS slot of the gdt.
	gdt[(GD_TSS0>>3)+cpunum()]=SEG16(STS_T32A,(uint32_t)&thiscpu->cpu_ts,
						sizeof(struct Taskstate),0);
	gdt[(GD_TSS0>>3)+cpunum()].sd_s=0;

	// Load the TSS selector (like other segment selectors, the
	// bottom three bits are special; we leave them 0)
	ltr(GD_TSS0+cpunum()*sizeof(struct Segdesc));

	// Load the IDT
	lidt(&idt_pd);
}
```
这部分近乎依葫芦画瓢即可，没什么好说的，thiscpu是个宏，每个cpu的结果不同。

但至此我们还不能启动其他CPU，还有一些兼容之前challenge的代码。在mp_main中，我们在加载kern_pgdir前要为每个cpu额外设置一次cr4来启用4MB页
```
#ifdef PSE_SUPPORT
	// enable optional 4MB page
	lcr4(rcr4()|CR4_PSE);
#endif

	lcr3(PADDR(kern_pgdir));
```
（忘记这个代码花了我很久来调试555，每个cpu都有自己的cr4寄存器）

至此，其他cpu可以正常启动。

既然有了多个cpu，接下来我们通过加锁来进行代码同步（之前因为每个其他cpu都通过for循环卡在那里，不会妨碍boot cpu的正常运行）
> Exercise 5. Apply the big kernel lock as described above, by calling lock_kernel() and unlock_kernel() at the proper locations.

关于加锁的原则，基本上来说就是每次进入内核且要访问公共资源（共享内存，但诸如每个cpu自己的内核栈这样的不算）之前加锁，最后一次访问后释放锁，因此：
1. 每一次调度前要加锁，因为调度需要访问全局的进程表。mpmain中：
    ```
    lock_kernel();

    sched_yield();
    ```
2. boot_aps前要加锁，这样可以确保在剩余的工作（创建第一个进程等等）完成前其他cpu能够卡在那里，因为它们在mp_main最后呼叫了调度器，会尝试锁一次。boot_aps前：
    ```
	// Acquire the big kernel lock before waking up APs
	// Your code here:
	lock_kernel();

	// Starting non-boot CPUs
	boot_aps();
    ```
3. 进入trap之前要加锁，但是仅当trap是由用户态触发时才加锁，因为内核态触发trap说明已经拿过锁了。trap中：
    ```
    if ((tf->tf_cs & 3) == 3) {
		// Trapped from user mode.
		// Acquire the big kernel lock before doing any
		// serious kernel work.
		// LAB 4: Your code here.
		assert(curenv);

		lock_kernel();

		// Garbage collect if current enviroment is a zombie
		if (curenv->env_status == ENV_DYING) {
			env_free(curenv);
			curenv = NULL;
			sched_yield();
		}

		// Copy trap frame (which is currently on the stack)
		// into 'curenv->env_tf', so that running the environment
		// will restart at the trap point.
		curenv->env_tf = *tf;
		// The trapframe on the stack should be ignored from here on.
		tf = &curenv->env_tf;
	}
    ```
4. （如果syscall使用sysenter/sysexit单独实现，那么）在syscall之前需要加锁，因为这必然是从用户态进入到内核态。syscall：
    ```
    // Dispatches to the correct kernel function, passing the arguments.
    int32_t
    syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
    {
        int ret;

        lock_kernel();
        ...
    }
    ```
那么何时释放锁呢？离开内核态前最后一次访问公共资源后，因此有两种情况：
1. 从trap中返回/env_run结束时，而它们都是使用一个相同的方法env_pop_tf退出的，因此在这个方法中：
    ```
    void
    env_pop_tf(struct Trapframe *tf)
    {
        // Record the CPU we are running on for user-space debugging
        curenv->env_cpunum = cpunum();

        unlock_kernel();

        asm volatile(
            "\tmovl %0,%%esp\n"
            "\tpopal\n"
            "\tpopl %%es\n"
            "\tpopl %%ds\n"
            "\taddl $0x8,%%esp\n" /* skip tf_trapno and tf_errcode */
            "\tiret\n"
            : : "g" (tf) : "memory");
        panic("iret failed");  /* mostly to placate the compiler */
    }
    ```
    注意这里一定要在最后一次写curenv这个公共资源后才能unlock
2. （如果syscall使用sysenter/sysexit单独实现，那么）在syscall退出之前需要释放锁。在syscall中：
    ```
        ...   
        unlock_kernel();

        return ret;
    }
    ```

Question
---
2. It seems that using the big kernel lock guarantees that only one CPU can run the kernel code at a time. Why do we still need separate kernel stacks for each CPU? Describe a scenario in which using a shared kernel stack will go wrong, even with the protection of the big kernel lock.

这个问题比较好想。因为任何一个CPU在进入内核态的时候都会破坏内核栈，因为进入内核态的那一瞬间就要在内核栈上保存当前用户态的cs、ip等等，而内核锁只是确保它们不同时进入临界区，无法禁止它们同时进入内核态，因此在目前我们的实现下必须每个cpu一个内核栈。那么能不能干脆禁止它们同时进入内核态呢？如果不引入新的硬件机制无法做到，因为锁是放在内核地址空间内的，在进入内核态之前无法访问。

解决了同步问题，我们实现一个调度器就可以开始多核执行了，接下来我们开始解决这部分。
> Exercise 6. Implement round-robin scheduling in sched_yield() as described above. Don't forget to modify syscall() to dispatch sys_yield().
> 
> Make sure to invoke sched_yield() in mp_main.
> 
> Modify kern/init.c to create three (or more!) environments that all run the program user/yield.c.
> 
> Run make qemu. You should see the environments switch back and forth between each other five times before terminating, like below.
> 
> Test also with several CPUS: make qemu CPUS=2.
> 
> ...
> Hello, I am environment 00001000.
> Hello, I am environment 00001001.
> Hello, I am environment 00001002.
> Back in environment 00001000, iteration 0.
> Back in environment 00001001, iteration 0.
> Back in environment 00001002, iteration 0.
> Back in environment 00001000, iteration 1.
> Back in environment 00001001, iteration 1.
> Back in environment 00001002, iteration 1.
> ...
> After the yield programs exit, there will be no runnable environment in the system, the scheduler should invoke the JOS kernel monitor. If any of this does not happen, then fix your code before proceeding.

首先，之前的代码没有把env做成环形结构，这里需要修正一下：
```
	// set next of final env to the first
	envs[NENV-1].env_link=envs;
```
接着是调度器的代码，比较简单，“旋转的罗宾”：
```
// Choose a user environment to run and run it.
void
sched_yield(void)
{
	struct Env *idle;

	// Implement simple round-robin scheduling.
	//
	// Search through 'envs' for an ENV_RUNNABLE environment in
	// circular fashion starting just after the env this CPU was
	// last running.  Switch to the first such environment found.
	//
	// If no envs are runnable, but the environment previously
	// running on this CPU is still ENV_RUNNING, it's okay to
	// choose that environment.
	//
	// Never choose an environment that's currently running on
	// another CPU (env_status == ENV_RUNNING). If there are
	// no runnable environments, simply drop through to the code
	// below to halt the cpu.

	// LAB 4: Your code here.

	int start,i;

	// start index of probing
	// if curenv exists, start from
	// its next, else start from 0
	start=curenv?curenv-envs+1:0;
	
	// search for runnable env
	for(i=0;i<NENV;i++)
		if(envs[(start+i)%NENV].env_status==ENV_RUNNABLE)
			break;

	// if new process is found runnable
	// run it
	if (i<NENV)
		env_run(&envs[(start+i)%NENV]);	

	// no new process found runnable

	// if current cpu has an process run and
	// it's still running(which means, it's 
	// still runnable), run it again
	if(curenv&&curenv->env_status==ENV_RUNNING)
		env_run(curenv);

	// current process is also feasible
	// halt the cpu

	// sched_halt never returns
	sched_halt();
}
```
这里用(start+i)%NENV计数是为了确保循环一圈

如果没有使用sysenter/sysexit来实现系统调用的话，至此就可以了，但是如果使用了的话需要在syscall前面再补充一部分工作，即保存当前用户的trapframe。

在之前没有调度器的时候，我们要么手动设置用户态的trapframe（如初始化第一个进程时），要么在由用户态进入内核态trap时立即保存而后使用env_pop_tf恢复，这两处都需要用到trapframe，而我们的系统调用不需要，因为sysenter/sysexit是使用寄存器来保存状态的而不是trapframe。引入了调度器后，我们就会在一个进程进入内核态后，切换到另一个进程接着运行，而目前我们的sysenter/sysexit实现缺乏对进入内核态的原进程的状态保存，也就是curenv的trapframe的保存，因此我们需要加上这一部分：
```
// Dispatches to the correct kernel function, passing the arguments.
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
	int ret;

	lock_kernel();

	// save trapframe of current environment
	// in curenv
	save_curenv_trapframe();

	// Call the function corresponding to the 'syscallno' parameter.
	// Return any appropriate return value.
	// LAB 3: Your code here.
	switch (syscallno) {
	case SYS_cputs:	
		sys_cputs((char*)a1,a2);
		ret=0;
		break;
	case SYS_cgetc: 
		ret=sys_cgetc();
		break;
	case SYS_getenvid: 
		ret=sys_getenvid();
		break;
	case SYS_env_destroy: 
		ret= sys_env_destroy(a1);
		break;
	case SYS_page_alloc:
		ret=sys_page_alloc(a1,(void*)a2,a3);
		break;
	case SYS_page_map:
		cprintf("0x%x\n",a5);
		ret=sys_page_map(a1,(void*)a2,a3,(void*)a4,a5);
		break;
	case SYS_page_unmap:
		ret=sys_page_unmap(a1,(void*)a2);
		break;
	case SYS_exofork:
		ret=sys_exofork();
		break;
	case SYS_env_set_status:
		ret=sys_env_set_status(a1,a2);
		break;
	case SYS_yield: 
		sys_yield();
		ret= 0;
		break;
	default:
		ret= -E_INVAL;
	}

	// save return value in env's trapframe
	// in case that this function doesn't return
	// and the env is resumed frorm its trapframe
	curenv->env_tf.tf_regs.reg_eax=ret;

	unlock_kernel();

	return ret;
}
```
包括两部分，一个是状态的保存，一个是返回值的保存，其中状态保存如下：
```
static inline 
void save_curenv_trapframe(){
	asm volatile("mov %%esi,%0":"=a"(curenv->env_tf.tf_eip));
	asm volatile("mov (%%ebp),%0":"=a"(curenv->env_tf.tf_esp));
	asm volatile("mov 0xc(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_edx));
	asm volatile("mov 0x10(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_ecx));
	asm volatile("mov 0x14(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_ebx));
	asm volatile("mov 0x18(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_edi));
	asm volatile("mov %%es,%0":"=a"(curenv->env_tf.tf_es));
	asm volatile("mov %%ds,%0":"=a"(curenv->env_tf.tf_ds));
}
```
没有对%esi进行保存，因为进入sysenter前它已经被保存了，现在的值没有用；没有保存%eax，后面要拿他来存放返回值;也没有保存cs,ss,eflags，目前够用了

同时注意寄存器状态保存必须要内联，这样可以使栈结构不被破坏，易于保存。

除此之外，我们对msr的初始化也要修改，并且将其加入到mp_main中，初始化每一个cpu的这部分寄存器。对msr初始化修改以适应多核：
```
static void msr_init(){
	extern void sysenter_handler();
	uint32_t cs;
	asm volatile("movl %%cs,%0":"=r"(cs));
	wrmsr(IA32_SYSENTER_CS,0x0,cs)
	wrmsr(IA32_SYSENTER_EIP,0x0,sysenter_handler)
	wrmsr(IA32_SYSENTER_ESP,0x0,KSTACKTOP-cpunum()*(KSTKGAP+KSTKSIZE)); // mp version
}
```
将其加入到mp_main:
```
// Setup code for APs
void
mp_main(void)
{
	// We are in high EIP now, safe to switch to kern_pgdir 
#ifdef PSE_SUPPORT
	// enable optional 4MB page
	lcr4(rcr4()|CR4_PSE);
#endif

	lcr3(PADDR(kern_pgdir));
	cprintf("SMP: CPU %d starting\n", cpunum());

	msr_init(); // init MSRs for this cpu as each cpu has its own separated registers
	lapic_init();
	env_init_percpu();
	trap_init_percpu();
	xchg(&thiscpu->cpu_status, CPU_STARTED); // tell boot_aps() we're up

	// Now that we have finished some basic setup, call sched_yield()
	// to start running processes on this CPU.  But make sure that
	// only one CPU can enter the scheduler at a time!
	//
	// Your code here:
	lock_kernel();

	sched_yield();
}
```
至此，可以正常运行。

Question
---
3. In your implementation of env_run() you should have called lcr3(). Before and after the call to lcr3(), your code makes references (at least it should) to the variable e, the argument to env_run. Upon loading the %cr3 register, the addressing context used by the MMU is instantly changed. But a virtual address (namely e) has meaning relative to a given address context--the address context specifies the physical address to which the virtual address maps. Why can the pointer e be dereferenced both before and after the addressing switch?
4. Whenever the kernel switches from one environment to another, it must ensure the old environment's registers are saved so they can be restored properly later. Why? Where does this happen?

问题3：e是在内核栈上的，虽然切换了进程，但每个进程都有一份内核内存的映射，因此即便切换也不影响我们访问

问题4：why就没啥好说的了，不保存怎么恢复？保存的发生时间刚才我们讨论过了，分两种情况，对于没有使用sysenter/sysexit的实现，就是在trap中保存的（当然前提是trap是从用户态发生的）；对于使用sysenter/sysexit的实现，是在我们进入syscall时手动保存的

有了多核支持，也有了调度系统，最后我们就是实现fork了。
> Exercise 7. Implement the system calls described above in kern/syscall.c and make sure syscall() calls them. You will need to use various functions in kern/pmap.c and kern/env.c, particularly envid2env(). For now, whenever you call envid2env(), pass 1 in the checkperm parameter. Be sure you check for any invalid system call arguments, returning -E_INVAL in that case. Test your JOS kernel with user/dumbfork and make sure it works before proceeding.

实现相对简单：
```
// Allocate a new environment.
// Returns envid of new environment, or < 0 on error.  Errors are:
//	-E_NO_FREE_ENV if no free environment is available.
//	-E_NO_MEM on memory exhaustion.
static envid_t
sys_exofork(void)
{
	// Create the new environment with env_alloc(), from kern/env.c.
	// It should be left as env_alloc created it, except that
	// status is set to ENV_NOT_RUNNABLE, and the register set is copied
	// from the current environment -- but tweaked so sys_exofork
	// will appear to return 0.

	// LAB 4: Your code here.
	struct Env* e;

	// allocate empty env
	if(env_alloc(&e,curenv->env_id)<0)
		return -E_NO_FREE_ENV;

	// set up kernel virtual memeory
	if(setupkvm(e->env_pgdir)<0)
		return -E_NO_MEM;

	// mark child process as not runnable for the moment
	e->env_status=ENV_NOT_RUNNABLE;

	// child process has the same register states
	// except for %eax, which implies the return value,
	// 0 for child process
	e->env_tf=curenv->env_tf;
	e->env_tf.tf_regs.reg_eax=0;

	return e->env_id;
}
```
```
// Set envid's env_status to status, which must be ENV_RUNNABLE
// or ENV_NOT_RUNNABLE.
//
// Returns 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
//	-E_INVAL if status is not a valid status for an environment.
static int
sys_env_set_status(envid_t envid, int status)
{
	// Hint: Use the 'envid2env' function from kern/env.c to translate an
	// envid to a struct Env.
	// You should set envid2env's third argument to 1, which will
	// check whether the current environment has permission to set
	// envid's status.

	// LAB 4: Your code here.
	struct Env* e;

	// check status legitimacy
	if(status!=ENV_RUNNABLE&&status!=ENV_NOT_RUNNABLE)
		return -E_INVAL;
	
	// retrieve env with permission checked
	if(envid2env(envid,&e,1)<0)
		return -E_BAD_ENV;
	
	e->env_status=status;

	return 0;	
}
```
```
// Allocate a page of memory and map it at 'va' with permission
// 'perm' in the address space of 'envid'.
// The page's contents are set to 0.
// If a page is already mapped at 'va', that page is unmapped as a
// side effect.
//
// perm -- PTE_U | PTE_P must be set, PTE_AVAIL | PTE_W may or may not be set,
//         but no other bits may be set.  See PTE_SYSCALL in inc/mmu.h.
//
// Return 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
//	-E_INVAL if va >= UTOP, or va is not page-aligned.
//	-E_INVAL if perm is inappropriate (see above).
//	-E_NO_MEM if there's no memory to allocate the new page,
//		or to allocate any necessary page tables.
static int
sys_page_alloc(envid_t envid, void *va, int perm)
{
	// Hint: This function is a wrapper around page_alloc() and
	//   page_insert() from kern/pmap.c.
	//   Most of the new code you write should be to check the
	//   parameters for correctness.
	//   If page_insert() fails, remember to free the page you
	//   allocated!

	// LAB 4: Your code here.
	struct Env* e;
	struct PageInfo* pp;
	pte_t* pte;
	int err;

	// check va and perm legitimacy
	if((uint32_t)va>=UTOP
		||((uint32_t)va%PGSIZE)!=0
		||(perm|PTE_SYSCALL)!=PTE_SYSCALL)
		return -E_INVAL;

	// retrieve env with permission checked
	if(envid2env(envid,&e,1)<0)
		return -E_BAD_ENV;
	
	// allocate page
	if((pp=page_alloc(ALLOC_ZERO))==0)
		return -E_NO_MEM;
	
	// insert
	if((err=page_insert(e->env_pgdir,pp,va,perm|PTE_U|PTE_P))<0){
		// free allocated page before return with error
		page_free(pp);
		return err;
	}

	return 0;
}
```
```
// Map the page of memory at 'srcva' in srcenvid's address space
// at 'dstva' in dstenvid's address space with permission 'perm'.
// Perm has the same restrictions as in sys_page_alloc, except
// that it also must not grant write access to a read-only
// page.
//
// Return 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if srcenvid and/or dstenvid doesn't currently exist,
//		or the caller doesn't have permission to change one of them.
//	-E_INVAL if srcva >= UTOP or srcva is not page-aligned,
//		or dstva >= UTOP or dstva is not page-aligned.
//	-E_INVAL is srcva is not mapped in srcenvid's address space.
//	-E_INVAL if perm is inappropriate (see sys_page_alloc).
//	-E_INVAL if (perm & PTE_W), but srcva is read-only in srcenvid's
//		address space.
//	-E_NO_MEM if there's no memory to allocate any necessary page tables.
static int
sys_page_map(envid_t srcenvid, void *srcva,
	     envid_t dstenvid, void *dstva, int perm)
{
	// Hint: This function is a wrapper around page_lookup() and
	//   page_insert() from kern/pmap.c.
	//   Again, most of the new code you write should be to check the
	//   parameters for correctness.
	//   Use the third argument to page_lookup() to
	//   check the current permissions on the page.

	// LAB 4: Your code here.
	struct Env *srce,*dste;
	struct PageInfo* pp;
	pte_t *pte;
	int errno;
	
	// check basic permission legitimacy
	if((perm|PTE_SYSCALL)!=PTE_SYSCALL)
		return -E_INVAL;

	// retrieve envs with permission checked
	if(envid2env(srcenvid,&srce,1)<0||envid2env(dstenvid,&dste,1)<0)
		return -E_BAD_ENV;
	
	// check va arrange legitimacy
	// and page-aligned
	if((uint32_t)srcva>=UTOP
		||(uint32_t)dstva>=UTOP
		||((uint32_t)srcva%PGSIZE)!=0
		||((uint32_t)dstva%PGSIZE)!=0)
		return -E_INVAL;
	
	// check src already has mapping for srcva
	if((pp=page_lookup(srce->env_pgdir,srcva,&pte))==0)
		return -E_INVAL;
	
	// verify that src grant write
	// permission only when it is granted that
	if(((~*pte)&PTE_W)&&(perm&PTE_W))
		return -E_INVAL;
	
	// perform mapping
	if((errno=page_insert(dste->env_pgdir,pp,dstva,perm|PTE_U|PTE_P))<0)
		return errno;

	return 0;
}
```
```
// Unmap the page of memory at 'va' in the address space of 'envid'.
// If no page is mapped, the function silently succeeds.
//
// Return 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
//	-E_INVAL if va >= UTOP, or va is not page-aligned.
static int
sys_page_unmap(envid_t envid, void *va)
{
	// Hint: This function is a wrapper around page_remove().

	// LAB 4: Your code here.
	struct Env* e;

	// check va legitimacy and page-aligned
	if((uint32_t)va>=UTOP||((uint32_t)va%PGSIZE)!=0)
		return -E_INVAL;

	// retrieve env with permission checked
	if(envid2env(envid,&e,1)<0)
		return -E_BAD_ENV;

	page_remove(e->env_pgdir,va);

	return 0;	
}
```
另外对于sysenter/sysexit实现系统调用需要修改一下用户态的sys_exofork:
```
// This must be inlined.  Exercise for reader: why?
static inline envid_t __attribute__((always_inline))
sys_exofork(void)
{
	envid_t ret;

	// save %ebp on stack to restore it
	// when child process returns
	asm volatile("pushl %ebp");
	// save user space %esp in %ebp passed into sysenter_handler
	asm volatile("movl %esp,%ebp");

	// save user space %eip in %esi passed into sysenter_handler
	asm volatile("leal .L%=,%%esi\n\t"
				 "sysenter\n\t"
				 ".L%=:"
			:
			: "a" (SYS_exofork)
			: "%esi","memory");
	
	// retrieve return value
	asm volatile("movl %%eax,%0":"=r"(ret));

	// restore %ebp
	asm volatile("popl %ebp");
	
	return ret;
}
```
这里代码中提出了一个问题，为什么sys_exofork必须要内联？这个我想了很长一会儿也没有想明白，睡了个午觉突然就明白了：是因为对fork的拆分。在sys_exofork设置了子进程的栈指针esp后，我们切回了用户态，经过诸多行为而后才完成对子进程内存的拷贝。这样会带来一个什么问题呢？如果sys_exofork中设置的栈指针的下面被用户态使用没关系，但如果栈指针的上面被用户态使用会造成栈损坏。也就是说我们必须确保从sys_exofork由内核态回到用户态直到子进程拷贝完成，用户态esp不会增长到拷贝时esp的上面，否则这部分栈结构会被破坏，毕竟子进程开始执行是按照拷贝时的esp来的。如果不内联，调用sys_exofork会使esp降低，使得拷贝给子进程的esp是较低值，返回用户态接着函数返回诸多内容出栈，esp升高，那么这时的esp与拷贝时esp之间的这段落差，会被push操作以及函数调用操作破坏掉。为了避免这一情况的发生，必须内联sys_exofork，使其拷贝给子进程esp时esp已经达到最高值，以后直到子进程拷贝完成前的所有操作都不会时esp上升到那时的esp的上面，也就确保了用户栈内容不被破坏。而如果fork没有被拆分，所有工作都在内核态一次完成，也就没有这个问题了。

但是至此有一个遗留问题没有解决，就是第五个参数的传递，之前是直接置0，尚且没有影响，现在不行了，我们改用用户栈来保存和传递。首先在sysenter之前，我们略微修改下栈结构：
```
	// save user space %esp in %ebp passed into sysenter_handler
	// also save the fifth parameter on the stack cuz there is no
	// idle register to pass it
	asm volatile("pushl %ebp");
	asm volatile("pushl %0"::"S"(a5));
	asm volatile("movl %esp,%ebp");
	asm volatile("add $4,%ebp");
```
也就是我们多push一个参数但又不改变ebp的位置，然后在，在trapentry，我们计算出它在栈上的位置然后push：
```
	# prepare arguments for syscall
	pushl -4(%ebp) # only four parameters supported to pass via register
				   # pass the fifth via user stack
	pushl %edi # arg4
	pushl %ebx # arg3
	pushl %ecx # arg2
	pushl %edx # arg1
	pushl %eax # sysno
	call syscall
```
如此一来我们就在不改变栈结构的基础上多输入了一个参数，只不过这个参数是通过访问用户栈来访问到的，这样五个参数就齐了

这样，fork也就实现完了。