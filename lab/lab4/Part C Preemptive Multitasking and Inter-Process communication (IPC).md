# Part C: Preemptive Multitasking and Inter-Process communication (IPC)

这一部分我们处理一下外部设备中断和IPC

外部设备中断
---
此前我们已经解决了内部异常，接下来我们就开始处理外部设备的硬件中断
> Exercise 13. Modify kern/trapentry.S and kern/trap.c to initialize the appropriate entries in the IDT and provide handlers for IRQs 0 through 15. Then modify the code in env_alloc() in kern/env.c to ensure that user environments are always run with interrupts enabled.
>
> Also uncomment the sti instruction in sched_halt() so that idle CPUs unmask interrupts.
> 
> The processor never pushes an error code when invoking a hardware interrupt handler. You might want to re-read section 9.2 of the 80386 Reference Manual, or section 5.8 of the IA-32 Intel Architecture Software Developer's Manual, Volume 3, at this time.
>
> After doing this exercise, if you run your kernel with any test program that runs for a non-trivial length of time (e.g., spin), you should see the kernel print trap frames for hardware interrupts. While interrupts are now enabled in the processor, JOS isn't yet handling them, so you should see it misattribute each interrupt to the currently running user environment and destroy it. Eventually it should run out of environments to destroy and drop into the monitor.

依然是在trap_init等等中注册中断向量：
在trapentry.S中：
```
/*
 * Lab 3: Your code here for generating entry points for the different traps.
 */
 .global int_handlers
 int_handlers:
 TRAPHANDLER_NOEC(divide,0)
 TRAPHANDLER_NOEC(debug,1)
 TRAPHANDLER_NOEC(nmi,2)
 TRAPHANDLER_NOEC(brkpt,3)
 TRAPHANDLER_NOEC(pflow,4)
 TRAPHANDLER_NOEC(_bound,5)
 TRAPHANDLER_NOEC(illop,6)
 TRAPHANDLER_NOEC(device,7)
 TRAPHANDLER(dblflt,8)
 TRAPHANDLER_NOEC(coproc,9)
 TRAPHANDLER(tss,10)
 TRAPHANDLER(segnp,11)
 TRAPHANDLER(stack,12)
 TRAPHANDLER(gpflt,13)
 TRAPHANDLER(pgflt,14)
 TRAPHANDLER(_res,15)
 TRAPHANDLER_NOEC(fperr,16)
 TRAPHANDLER(_align,17)
 TRAPHANDLER_NOEC(mchk,18)
 TRAPHANDLER_NOEC(simderr,19)
 TRAPHANDLER_NOEC(irq0,IRQ_OFFSET+0)
 TRAPHANDLER_NOEC(irq1,IRQ_OFFSET+1)
 TRAPHANDLER_NOEC(irq2,IRQ_OFFSET+2)
 TRAPHANDLER_NOEC(irq3,IRQ_OFFSET+3)
 TRAPHANDLER_NOEC(irq4,IRQ_OFFSET+4)
 TRAPHANDLER_NOEC(irq5,IRQ_OFFSET+5)
 TRAPHANDLER_NOEC(irq6,IRQ_OFFSET+6)
 TRAPHANDLER_NOEC(irq7,IRQ_OFFSET+7)
 TRAPHANDLER_NOEC(irq8,IRQ_OFFSET+8)
 TRAPHANDLER_NOEC(irq9,IRQ_OFFSET+9)
 TRAPHANDLER_NOEC(irq10,IRQ_OFFSET+10)
 TRAPHANDLER_NOEC(irq11,IRQ_OFFSET+11)
 TRAPHANDLER_NOEC(irq12,IRQ_OFFSET+12)
 TRAPHANDLER_NOEC(irq13,IRQ_OFFSET+13)
 TRAPHANDLER_NOEC(irq14,IRQ_OFFSET+14)
 TRAPHANDLER_NOEC(irq15,IRQ_OFFSET+15)
```
而后在trap_init中：
```
	// LAB 3: Your code here.
	extern long int_handlers[][4];
	for(int i=0;i<20;i++)
		SETGATE(idt[i],0,GD_KT,int_handlers[i],0);
	SETGATE(idt[T_BRKPT],0,GD_KT,int_handlers[T_BRKPT],3);

	for(int i=0;i<=15;i++)
		SETGATE(idt[IRQ_OFFSET+i],0,GD_KT,int_handlers[T_SIMDERR+1+i],0);
```
这里也修复了之前的一个bug，SETGATE中第二个位置定义为：
```
#define SETGATE(gate, istrap, sel, off, dpl)			\
{								\
	(gate).gd_off_15_0 = (uint32_t) (off) & 0xffff;		\
	(gate).gd_sel = (sel);					\
	(gate).gd_args = 0;					\
	(gate).gd_rsv1 = 0;					\
	(gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;	\
	(gate).gd_s = 0;					\
	(gate).gd_dpl = (dpl);					\
	(gate).gd_p = 1;					\
	(gate).gd_off_31_16 = (uint32_t) (off) >> 16;		\
}
```
这里应当设置为0使得istrap==0，设置为中断门，使得进入exception和interrupt时都关中断（因为这里要求进入trap处理exception时必须assert中断的关闭）

以及对其他核halt时的中断set:
```
// Halt this CPU when there is nothing to do. Wait until the
// timer interrupt wakes it up. This function never returns.
//
void
sched_halt(void)
{
	int i;

	// For debugging and testing purposes, if there are no runnable
	// environments in the system, then drop into the kernel monitor.
	for (i = 0; i < NENV; i++) {
		if ((envs[i].env_status == ENV_RUNNABLE ||
		     envs[i].env_status == ENV_RUNNING ||
		     envs[i].env_status == ENV_DYING))
			break;
	}
	if (i == NENV) {
		cprintf("No runnable environments in the system!\n");
		while (1)
			monitor(NULL);
	}

	// Mark that no environment is running on this CPU
	curenv = NULL;
	lcr3(PADDR(kern_pgdir));

	// Mark that this CPU is in the HALT state, so that when
	// timer interupts come in, we know we should re-acquire the
	// big kernel lock
	xchg(&thiscpu->cpu_status, CPU_HALTED);

	// Release the big kernel lock as if we were "leaving" the kernel
	unlock_kernel();

	// Reset stack pointer, enable interrupts and then halt.
	asm volatile (
		"movl $0, %%ebp\n"
		"movl %0, %%esp\n"
		"pushl $0\n"
		"pushl $0\n"
		// Uncomment the following line after completing exercise 13
		"sti\n"
		"1:\n"
		"hlt\n"
		"jmp 1b\n"
	: : "a" (thiscpu->cpu_ts.ts_esp0));
}
```
为了使得当运行在用户态时中断总是打开，我们需要在进程分配时就打开中断位：

env_alloc中：
```
	// Enable interrupts while in user mode.
	// LAB 4: Your code here.
	e->env_tf.tf_eflags|=FL_IF;
```

接下来我们处理时钟中断：
> Exercise 14. Modify the kernel's trap_dispatch() function so that it calls sched_yield() to find and run a different environment whenever a clock interrupt takes place.
> 
> You should now be able to get the user/spin test to work: the parent environment should fork off the child, sys_yield() to it a couple times but in each case regain control of the CPU after one time slice, and finally kill the child environment and terminate gracefully.

这一步比较简单，按照指引即可：
```
	// Handle clock interrupts. Don't forget to acknowledge the
	// interrupt using lapic_eoi() before calling the scheduler!
	// LAB 4: Your code here.
	if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
		lapic_eoi();
		sched_yield();
	}
```

至此，不使用sysenter/sysexit的实现下已经可以正常运行，但使用了的话还需要其他兼容代码。

首先需要添加对eflag寄存器的保存和恢复，在lib/syscall.c和inc/lib.h中：
```
static int32_t
syscall(int num, int check, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
	int32_t ret;
	// Generic system call: pass system call number in AX,
	// up to five parameters in DX, CX, BX, DI, SI.
	// Interrupt kernel with T_SYSCALL.
	//
	// The "volatile" tells the assembler not to optimize
	// this instruction away just because we don't use the
	// return value.
	//
	// The last clause tells the assembler that this can
	// potentially change the condition codes and arbitrary
	// memory locations.

	// move arguments into registers before %ebp modified
	// as with static identifier, num, check and a1 are stroed
	// in %eax, %edx and %ecx respectively, while a3, a4 and a5
	// are addressed via 0xx(%ebp)
	asm volatile("movl %0,%%edx"::"S"(a1):"%edx");
	asm volatile("movl %0,%%ecx"::"S"(a2):"%ecx");
	asm volatile("movl %0,%%ebx"::"S"(a3):"%ebx");
	asm volatile("movl %0,%%edi"::"S"(a4):"%ebx");

	// 1. save eflags
	// 2. save user space %esp in %ebp passed into sysenter_handler
	// 3. save the fifth parameter on the stack cuz there is no
	// idle register to pass it
	asm volatile("pushfl");
	asm volatile("pushl %ebp");
	asm volatile("pushl %0"::"S"(a5));
	asm volatile("add $4,%esp");
	asm volatile("movl %esp,%ebp");

	// save user space %eip in %esi passed into sysenter_handler
	asm volatile("leal .syslabel,%%esi":::"%esi");
	asm volatile("sysenter \n\t"
				 ".syslabel:"
			:
			: "a" (num)
			: "memory");
	
	// retrieve return value
	asm volatile("movl %%eax,%0":"=r"(ret));
	
	// restore %ebp and shift for eflags
	asm volatile("popl %ebp");
	asm volatile("add $0x4,%esp");

	if(check && ret > 0)
		panic("syscall %d returned %d (> 0)", num, ret);
	return ret;
}
```
```
// This must be inlined.  Exercise for reader: why?
static inline envid_t __attribute__((always_inline))
sys_exofork(void)
{
	envid_t ret;

	// save eflags
	// save %ebp on stack to restore it
	// when child process returns
	asm volatile("pushfl");
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

	// restore %ebp and shift for eflags
	asm volatile("popl %ebp");
	asm volatile("add $0x4,%esp");
	
	return ret;
}
```
都是在栈上保存ebp之前pushf，系统调用结束后恢复ebp后add esp 4来跳过这个eflags。这里并没有popf，主要是不需要。因为进入系统调用后有两种方式回到原进程:
1. 系统调用自己回来
2. 通过sys_yield后的env_run回来

后者在恢复时直接使用tf中的信息popf所以不需要我们pop，而对后者而言我们的pushf仅仅是为了能够让内核态读取在栈上找到我们的eflags并保存在curenv->tf中，并不是为了我们自己恢复，而且用户态popf时无法set IF位，用户态自行恢复也是不现实的

对于前者，我们的pushf目的也是相同的，是为了能够让内核态找到我们的eflags。sysenter在进入内核态时会清空IF位：
> Operation
> When SYSENTER is called, CS is set to the value in IA32_SYSENTER_CS. SS is set to IA32_SYSENTER_CS + 8. EIP is loaded from IA32_SYSENTER_EIP and ESP is loaded from IA32_SYSENTER_ESP. The CPU is now in ring 0, with EFLAGS.IF=0, EFLAGS.VM=0, EFLAGS.RF=0.
> 
> When SYSEXIT is called, CS is set to IA32_SYSENTER_CS+16. EIP is set to EDX. SS is set to IA32_SYSENTER_CS+24, and ESP is set to ECX.
> 
> Notes: ECX and EDX are not automatically saved as the return address and Stack Pointer. These need to be saved in Ring 3.

（可以参考[wiki](https://wiki.osdev.org/SYSENTER)）

但sysexit并不会恢复eflags，因此我们要手动恢复。首先在syscall中保存eflags:
```
static inline 
void save_curenv_trapframe(){
	asm volatile("mov %%esi,%0":"=a"(curenv->env_tf.tf_eip));
	asm volatile("mov (%%ebp),%0":"=a"(curenv->env_tf.tf_esp));
	asm volatile("mov (%1),%0":"=a"(curenv->env_tf.tf_regs.reg_ebp):"a"(curenv->env_tf.tf_esp));
	asm volatile("mov 4(%1),%0":"=a"(curenv->env_tf.tf_eflags):"a"(curenv->env_tf.tf_esp));
	asm volatile("mov 0xc(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_edx));
	asm volatile("mov 0x10(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_ecx));
	asm volatile("mov 0x14(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_ebx));
	asm volatile("mov 0x18(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_edi));
	asm volatile("mov %%es,%0":"=a"(curenv->env_tf.tf_es));
	asm volatile("mov %%ds,%0":"=a"(curenv->env_tf.tf_ds));
}
```
而后在离开内核态后恢复：
```
// Dispatches to the correct kernel function, passing the arguments.
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
	int ret;
	uint32_t eflags;

	lock_kernel();

	// save trapframe of current environment
	// in curenv
	save_curenv_trapframe();
	
    ...

	// save return value in env's trapframe
	// in case that this function doesn't return
	// and the env is resumed frorm its trapframe
	curenv->env_tf.tf_regs.reg_eax=ret;

	// read user eflags to restore it manually
	// Here we must read and cache it on kernel
	// stack before unlock kernel cuz curenv is
	// a public memory resource, we are fobidden 
	// to read it wihout holding kernel lock
	eflags=curenv->env_tf.tf_eflags;

	unlock_kernel();
	
	// restore user eflags with IF diabled
	// (later enable it right before 'sysexit' instruction)
	write_eflags(eflags&~FL_IF);

	return ret;
}
```
注意这里用户的eflags必须要在释放内核锁之后写入，防止unlock_kernel过程中修改，但读取curenv的eflags读取必须要在拿着内核锁的状态下进行，因为这个内存是公共访问内存，所以就用局部变量临时缓存在当前cpu的内核栈上，用于写入。这里还有一点，必须要先clear FI_IF位再恢复，因为打开中断与退出内核态必须是原子操作，下面来解释如何进行。在trapentry.S中：
```
.global sysenter_handler
 sysenter_handler:

	# prepare arguments for syscall
	pushl -4(%ebp) # only four parameters supported to pass via register
				   # pass the fifth via user stack
	pushl %edi # arg4
	pushl %ebx # arg3
	pushl %ecx # arg2
	pushl %edx # arg1
	pushl %eax # sysno
	call syscall

	# pepare user space information
	# to restore
	movl %esi,%edx # return eip
	movl %ebp,%ecx # user space esp

	# before returning to userspace
	# enable the interrupt
	sti
	sysexit
```
在执行sysexit指令之前设置IF位，这里依旧是原子操作，因为sti指令后下一个指令与sti之间仍然是无法触发中断的：
> ##  Description：
> 
> In most cases, STI sets the interrupt flag (IF) in the EFLAGS register. This allows the processor to respond to maskable hardware interrupts.
> 
> If IF = 0, maskable hardware interrupts remain inhibited on the instruction boundary following an execution of STI. (The delayed effect of this instruction is provided to allow interrupts to be enabled just before returning from a procedure or subroutine. For instance, if an STI instruction is followed by an RET instruction, the RET instruction is allowed to execute before external interrupts are recognized. No interrupts can be recognized if an execution of CLI immediately follow such an execution of STI.) The inhibition ends after delivery of another event (e.g., exception) or the execution of the next instruction.

可以参考[这里](https://www.felixcloutier.com/x86/sti)

如此一来，我们就在使用sysenter/sysexit的情况下实现了对用户态寄存器的保存和恢复，至此，程序可以正常运行

进程间通信
---
进程间通信比之前简单许多，也好理解很多，只是安全检查比较麻烦
> Exercise 15. Implement sys_ipc_recv and sys_ipc_try_send in kern/syscall.c. Read the comments on both before implementing them, since they have to work together. When you call envid2env in these routines, you should set the checkperm flag to 0, meaning that any environment is allowed to send IPC messages to any other environment, and the kernel does no special permission checking other than verifying that the target envid is valid.
>
> Then implement the ipc_recv and ipc_send functions in lib/ipc.c.
> 
> Use the user/pingpong and user/primes functions to test your IPC mechanism. user/primes will generate for each prime number a new environment until JOS runs out of environments. You might find it interesting to read user/primes.c to see all the forking and IPC going on behind the scenes.

kernel部分代码：
```
// Try to send 'value' to the target env 'envid'.
// If srcva < UTOP, then also send page currently mapped at 'srcva',
// so that receiver gets a duplicate mapping of the same page.
//
// The send fails with a return value of -E_IPC_NOT_RECV if the
// target is not blocked, waiting for an IPC.
//
// The send also can fail for the other reasons listed below.
//
// Otherwise, the send succeeds, and the target's ipc fields are
// updated as follows:
//    env_ipc_recving is set to 0 to block future sends;
//    env_ipc_from is set to the sending envid;
//    env_ipc_value is set to the 'value' parameter;
//    env_ipc_perm is set to 'perm' if a page was transferred, 0 otherwise.
// The target environment is marked runnable again, returning 0
// from the paused sys_ipc_recv system call.  (Hint: does the
// sys_ipc_recv function ever actually return?)
//
// If the sender wants to send a page but the receiver isn't asking for one,
// then no page mapping is transferred, but no error occurs.
// The ipc only happens when no errors occur.
//
// Returns 0 on success, < 0 on error.
// Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist.
//		(No need to check permissions.)
//	-E_IPC_NOT_RECV if envid is not currently blocked in sys_ipc_recv,
//		or another environment managed to send first.
//	-E_INVAL if srcva < UTOP but srcva is not page-aligned.
//	-E_INVAL if srcva < UTOP and perm is inappropriate
//		(see sys_page_alloc).
//	-E_INVAL if srcva < UTOP but srcva is not mapped in the caller's
//		address space.
//	-E_INVAL if (perm & PTE_W), but srcva is read-only in the
//		current environment's address space.
//	-E_NO_MEM if there's not enough memory to map srcva in envid's
//		address space.
static int
sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
{
	// LAB 4: Your code here.
	struct Env* tarenv;
	pte_t *srcpte,*dstpte;
	struct PageInfo* pp;

	// retrieve target env
	if(envid2env(envid,&tarenv,0)<0)
		return -E_BAD_ENV;
	// assert target env is receiving
	if(!tarenv->env_ipc_recving||tarenv->env_status!=ENV_NOT_RUNNABLE)
		return -E_IPC_NOT_RECV;
	
	// whether receiver is receiving a page mapping
	// if so, perform a series of securiy check of 
	// srcva and finally insert it to target env's
	// page mapping
	if((uint32_t)tarenv->env_ipc_dstva<UTOP){
		// verify srcva is legitimate and page-aligned
		if((uint32_t)srcva>=UTOP||(uint32_t)srcva%PGSIZE!=0)
			return -E_INVAL;
		// verify srcva resides in curenv's page mapping
		if((pp=page_lookup(curenv->env_pgdir,srcva,&srcpte))<0)
			return -E_INVAL;
		// check perm legitimacy
		if((perm|PTE_SYSCALL)!=PTE_SYSCALL)
			return -E_INVAL;
		// check write permission grant
		if((~*srcpte&PTE_W)&&(perm&PTE_W))
			return -E_INVAL;
		// perform page mapping insertion
		if(page_insert(tarenv->env_pgdir,pp,tarenv->env_ipc_dstva,perm)<0)
			return -E_NO_MEM;
	}

	// record informations
	tarenv->env_ipc_from=curenv->env_id;
	tarenv->env_ipc_value=value;

	// reset target environment to runnable state
	tarenv->env_ipc_recving=false;
	tarenv->env_status=ENV_RUNNABLE;

	// set return value for receiver
	tarenv->env_tf.tf_regs.reg_eax=0;

	return 0;
}

// Block until a value is ready.  Record that you want to receive
// using the env_ipc_recving and env_ipc_dstva fields of struct Env,
// mark yourself not runnable, and then give up the CPU.
//
// If 'dstva' is < UTOP, then you are willing to receive a page of data.
// 'dstva' is the virtual address at which the sent page should be mapped.
//
// This function only returns on error, but the system call will eventually
// return 0 on success.
// Return < 0 on error.  Errors are:
//	-E_INVAL if dstva < UTOP but dstva is not page-aligned.
static int
sys_ipc_recv(void *dstva)
{
	// first check whether expect to
	// receive a page mapping
	if((uint32_t)dstva<UTOP){
		// if so, verify dstva page-aligned
		if((uint32_t)dstva%PGSIZE!=0)
			return -E_INVAL;
		
		curenv->env_ipc_dstva=dstva;		
	}else
		// (necessarily)set the field above UTOP
		// rather than leave it 0x0
		// to inform sender it does not 
		// expect to receive a page mapping
		curenv->env_ipc_dstva=(void*)~0;		

	// mark itself waiting for ipc
	curenv->env_ipc_recving=true;
	curenv->env_status=ENV_NOT_RUNNABLE;

	sched_yield();

	// actually never reach here
	return 0;
}
```
lib部分代码：
```
// Receive a value via IPC and return it.
// If 'pg' is nonnull, then any page sent by the sender will be mapped at
//	that address.
// If 'from_env_store' is nonnull, then store the IPC sender's envid in
//	*from_env_store.
// If 'perm_store' is nonnull, then store the IPC sender's page permission
//	in *perm_store (this is nonzero iff a page was successfully
//	transferred to 'pg').
// If the system call fails, then store 0 in *fromenv and *perm (if
//	they're nonnull) and return the error.
// Otherwise, return the value sent by the sender
//
// Hint:
//   Use 'thisenv' to discover the value and who sent it.
//   If 'pg' is null, pass sys_ipc_recv a value that it will understand
//   as meaning "no page".  (Zero is not the right value, since that's
//   a perfectly valid place to map a page.)
int32_t
ipc_recv(envid_t *from_env_store, void *pg, int *perm_store)
{
	// LAB 4: Your code here.
	int r;
	
	// system call to receive information
	// if fail, set two pointers to zero
	// and return 
	if((r=sys_ipc_recv(pg?pg:(void*)~0))<0){
		if(from_env_store)
			*from_env_store=0;
		if(perm_store)
			*perm_store=0;
		return r;
	}

	// successfully receive
	// update pointer if demanded
	if(from_env_store)
		*from_env_store=thisenv->env_ipc_from;
	if(perm_store)
		*perm_store=thisenv->env_ipc_perm;

	// return ipc value
	return thisenv->env_ipc_value;
}

// Send 'val' (and 'pg' with 'perm', if 'pg' is nonnull) to 'toenv'.
// This function keeps trying until it succeeds.
// It should panic() on any error other than -E_IPC_NOT_RECV.
//
// Hint:
//   Use sys_yield() to be CPU-friendly.
//   If 'pg' is null, pass sys_ipc_try_send a value that it will understand
//   as meaning "no page".  (Zero is not the right value.)
void
ipc_send(envid_t to_env, uint32_t val, void *pg, int perm)
{
	// LAB 4: Your code here.
	int r;
	while((r=sys_ipc_try_send(to_env,val,pg?pg:(void*)~0,perm))!=0){
		// before retrying first
		// yield to be CPU friendly
		if(r==-E_IPC_NOT_RECV){
			sys_yield();
			continue;
		}else
			return;
	}
}
```
以及在syscall中添加相应dispatch项：
```
    ...
	case SYS_ipc_try_send:
		ret=sys_ipc_try_send(a1,a2,(void*)a3,a4);
		break;
	case SYS_ipc_recv:
		ret=sys_ipc_recv((void*)a1);
		break;
    ...
```
需要注意的是为了对cpu友好（也为了提高send try的成功率），每次send try失败都重新调度一次。

最后是修复我个人的一个bug，在sched中：
```
	// if new process is found runnable
	// run it
	if (i<NENV){
		// if curenv is still running, mark it 
		// runnable to allow it rerunning
		if(curenv&&curenv->env_status==ENV_RUNNING)
			curenv->env_status=ENV_RUNNABLE;
		
		env_run(&envs[(start+i)%NENV]);	
	}
```
运行新选定的进程前，如果当前进程仍然是running状态，那说明它仍然runnable，应当更新其状态

至此，所以代码ok
