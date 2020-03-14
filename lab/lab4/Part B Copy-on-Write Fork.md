# Part B: Copy-on-Write Fork

这部分的主要内容是延续上一部分的fork，实现copy-on-write的高效版本fork

总的来说分两部分：处理页面错误和实现fork

Page fault handler
---
对copy-on-write，我们首先要面对的问题就是，如何处理页面错误？这里给出的方案是，在用户态的库函数中实现handler，通过系统调用注册，发生页面错误是dispatch到handler，而后回到进程暂停的地方，总的来说类似于alarmtest中注册的时钟函数。

就像注册时钟函数那样，我们需要在用户态提供注册的handler外面套个wrapper来处理，这个我们放到后面，先来实现注册handler的系统调用，再谈wrapper

> Exercise 8. Implement the sys_env_set_pgfault_upcall system call. Be sure to enable permission checking when looking up the environment ID of the target environment, since this is a "dangerous" system call.

```
// Set the page fault upcall for 'envid' by modifying the corresponding struct
// Env's 'env_pgfault_upcall' field.  When 'envid' causes a page fault, the
// kernel will push a fault record onto the exception stack, then branch to
// 'func'.
//
// Returns 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
static int
sys_env_set_pgfault_upcall(envid_t envid, void *func)
{
	// LAB 4: Your code here.
	struct Env* e;

	// check envid permission
	if(envid2env(envid,&e,1)<0)
		return -E_BAD_ENV;
	
	e->env_pgfault_upcall=func;

	return 0;
}
```
以及在syscall中追加一个case：
```
    ...
	
    case SYS_env_set_pgfault_upcall:
		ret=sys_env_set_pgfault_upcall(a1,(void*)a2);
		break;
    
    ...
```
这个很简单，没啥好说的

接下来我们要实现的是内核态的行为。内核在接受到一个用户态页错误的时候，应当在“用户异常栈”上准备一个UTrapframe供handler使用，然后跳转到handler，同时将esp指向这个“用户异常栈”的UTrapframe。这个“用户异常栈”就是专门用来处理页面错误的，在内核的这部分代码中，我们总是假定这个栈是已经准备好的（由用户态库函数使用系统调用来分配在固定位置）

```
void
page_fault_handler(struct Trapframe *tf)
{
	uint32_t fault_va;

	// Read processor's CR2 register to find the faulting address
	fault_va = rcr2();

	// Handle kernel-mode page faults.
	// LAB 3: Your code here.
	if(!(tf->tf_cs&0x3))
		panic("Kernel page fault\n");
	
	// We've already handled kernel-mode exceptions, so if we get here,
	// the page fault happened in user mode.

	// Call the environment's page fault upcall, if one exists.  Set up a
	// page fault stack frame on the user exception stack (below
	// UXSTACKTOP), then branch to curenv->env_pgfault_upcall.
	//
	// The page fault upcall might cause another page fault, in which case
	// we branch to the page fault upcall recursively, pushing another
	// page fault stack frame on top of the user exception stack.
	//
	// It is convenient for our code which returns from a page fault
	// (lib/pfentry.S) to have one word of scratch space at the top of the
	// trap-time stack; it allows us to more easily restore the eip/esp. In
	// the non-recursive case, we don't have to worry about this because
	// the top of the regular user stack is free.  In the recursive case,
	// this means we have to leave an extra word between the current top of
	// the exception stack and the new stack frame because the exception
	// stack _is_ the trap-time stack.
	//
	// If there's no page fault upcall, the environment didn't allocate a
	// page for its exception stack or can't write to it, or the exception
	// stack overflows, then destroy the environment that caused the fault.
	// Note that the grade script assumes you will first check for the page
	// fault upcall and print the "user fault va" message below if there is
	// none.  The remaining three checks can be combined into a single test.
	//
	// Hints:
	//   user_mem_assert() and env_run() are useful here.
	//   To change what the user environment runs, modify 'curenv->env_tf'
	//   (the 'tf' variable points at 'curenv->env_tf').

	// LAB 4: Your code here.
	struct PageInfo* pp;
	struct UTrapframe* utf;
	
	// non-null page fault handler
	if(curenv->env_pgfault_upcall!=0){
		// assert function address legitimacy
		user_mem_assert(curenv,(void*)curenv->env_pgfault_upcall,PGSIZE,0);

		// create a new trapframe 		

		// the following condition incidates that 
		// this is a recursive page fault
		// when %esp is [UXSTACKTOP-PGSIZE,UXSTACKTOP)
		if(tf->tf_esp>=(UXSTACKTOP-PGSIZE)
				&&tf->tf_esp<UXSTACKTOP)
			// dedicate a word to accommodate eip 
			utf=(struct UTrapframe*)(tf->tf_esp-sizeof(struct UTrapframe)-4);
		else
			utf=(struct UTrapframe*)(UXSTACKTOP-sizeof(struct UTrapframe));

		// assert allocated utf is legitimate
		user_mem_assert(curenv,(void*)utf,sizeof(struct UTrapframe),PTE_W);

		// prepare exception trapframe
		utf->utf_eflags=curenv->env_tf.tf_eflags;
		utf->utf_eip=curenv->env_tf.tf_eip;
		utf->utf_err=curenv->env_tf.tf_err;
		utf->utf_esp=curenv->env_tf.tf_esp;
		utf->utf_fault_va=fault_va;
		utf->utf_regs=curenv->env_tf.tf_regs;

		// make userspace run in handler 
		// and esp point to exception stack 
		// when it resumes
		curenv->env_tf.tf_eip=(uintptr_t)curenv->env_pgfault_upcall;
		curenv->env_tf.tf_esp=(uintptr_t)utf;

		env_run(curenv);		
	}
	
destroy:
	// Destroy the environment that caused the fault.
	cprintf("[%08x] user fault va %08x ip %08x\n",
		curenv->env_id, fault_va, tf->tf_eip);
	print_trapframe(tf);
	env_destroy(curenv);
}
```
根据进程的handler指针是否为空来决定要不要处理，同时检查合法性。处理时，如果是嵌套异常（处理页错误时发生页错误），就在栈上接着向下挪一个4字节和一个UTrapframe的大小然后分配，如果不是嵌套，就直接在从栈顶往下开始分配。这里嵌套异常多挪一个4字节会在后面wrapper介绍，先按下不表。其余的就都是常规操作了，修改进程tf的eip指针来修改其resume时的位置，通过修改esp使其指向异常栈。

但是我们不禁想问，那handler返回时怎么恢复到原来的栈呢？毕竟ret时不会切换sp啊。这时候就是wrapper的意义之所在了。
> Exercise 10. Implement the _pgfault_upcall routine in lib/pfentry.S. The interesting part is returning to the original point in the user code that caused the page fault. You'll return directly there, without going back through the kernel. The hard part is simultaneously switching stacks and re-loading the EIP.

总的来说，我们需要在原先的用户栈上放返回地址，这样我们就可以先切换esp回到原来用户栈，同时栈上还是放着返回地址，这也解释了为什么嵌套页错误时需要多预留一个4字节了，是为了存放返回地址以便切换esp后栈顶仍然放着resume address
```
.text
.globl _pgfault_upcall
_pgfault_upcall:
	// Call the C page fault handler.
	pushl %esp			// function argument: pointer to UTF
	movl _pgfault_handler, %eax
	call *%eax
	addl $4, %esp			// pop function argument
	
	// Now the C page fault handler has returned and you must return
	// to the trap time state.
	// Push trap-time %eip onto the trap-time stack.
	//
	// Explanation:
	//   We must prepare the trap-time stack for our eventual return to
	//   re-execute the instruction that faulted.
	//   Unfortunately, we can't return directly from the exception stack:
	//   We can't call 'jmp', since that requires that we load the address
	//   into a register, and all registers must have their trap-time
	//   values after the return.
	//   We can't call 'ret' from the exception stack either, since if we
	//   did, %esp would have the wrong value.
	//   So instead, we push the trap-time %eip onto the *trap-time* stack!
	//   Below we'll switch to that stack and call 'ret', which will
	//   restore %eip to its pre-fault value.
	//
	//   In the case of a recursive fault on the exception stack,
	//   note that the word we're pushing now will fit in the
	//   blank word that the kernel reserved for us.
	//
	// Throughout the remaining code, think carefully about what
	// registers are available for intermediate calculations.  You
	// may find that you have to rearrange your code in non-obvious
	// ways as registers become unavailable as scratch space.
	//
	// LAB 4: Your code here.
	movl 0x30(%esp),%eax # trap-time esp
	movl 0x28(%esp),%edx # trap-time eip
	sub $0x4,%eax # new trap-time esp
	movl %edx,(%eax) # save return addr on trap-time stack 
	movl %eax,0x30(%esp) # update esp to switch
	add $0x8,%esp # bypass fault_va and err

	// Restore the trap-time registers.  After you do this, you
	// can no longer modify any general-purpose registers.
	// LAB 4: Your code here.
	popal
	add $0x4,%esp # by pass eip

	// Restore eflags from the stack.  After you do this, you can
	// no longer use arithmetic operations or anything else that
	// modifies eflags.
	// LAB 4: Your code here.
	popfl

	// Switch back to the adjusted trap-time stack.
	// LAB 4: Your code here.
	popl %esp # stack switch

	// Return to re-execute the instruction that faulted.
	// LAB 4: Your code here.
	ret
```
也因此，每个进程env结构体中的函数指针都相同，只不过每个进程的全局变量_pgfault_handler不同罢了，注册时我们要设置的其实也是这个全局变量
> Exercise 11. Finish set_pgfault_handler() in lib/pgfault.c.
```
//
// Set the page fault handler function.
// If there isn't one yet, _pgfault_handler will be 0.
// The first time we register a handler, we need to
// allocate an exception stack (one page of memory with its top
// at UXSTACKTOP), and tell the kernel to call the assembly-language
// _pgfault_upcall routine when a page fault occurs.
//
void
set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
{
	if (_pgfault_handler == 0) {
		// First time through!
		// LAB 4: Your code here.

		// allocate page for exception stack
		if(sys_page_alloc(0,(void*)(UXSTACKTOP-PGSIZE),PTE_W|PTE_U)<0)
			panic("Fail to allocate page for exception stack");
		// set env's handler function pointer to non-null
		if(sys_env_set_pgfault_upcall(0,_pgfault_upcall)<0)
			panic("Fail to register page fault handler");
	}

	// Save handler pointer for assembly to call.
	_pgfault_handler = handler;
}
```
至此也没什么好说的了

Copy-on-write fork
---
有了这些工具函数，我们开始实现copy-on-write fork

> ps:本人在此栽了很久，花了整整一天半什么都没干debug，真的痛苦至极。

> Exercise 12. Implement fork, duppage and pgfault in lib/fork.c.
>
> Test your code with the forktree program. It should produce the following messages, with interspersed 'new env', 'free env', and 'exiting gracefully' messages. The messages may not appear in this order, and the environment IDs may be different.
> 
>	1000: I am ''
> 
>	1001: I am '0'
> 
>	2000: I am '00'
> 
>	2001: I am '000'
> 
>	1002: I am '1'
> 
>	3000: I am '11'
> 
>	3001: I am '10'
> 
>	4000: I am '100'
> 
>	1003: I am '01'
> 
>	5000: I am '010'
> 
>	4001: I am '011'
> 
>	2002: I am '110'
> 
>	1004: I am '001'
> 
>	1005: I am '111'
> 
>	1006: I am '101'
	
我们按照自底向上，先实现handler。
```
//
// Custom page fault handler - if faulting page is copy-on-write,
// map in our own private writable copy.
//
static void
pgfault(struct UTrapframe *utf)
{
	extern volatile pte_t uvpt[];
	void *addr = (void *) utf->utf_fault_va;
	uint32_t err = utf->utf_err;
	void *tmp,*origin;
	
	// Check that the faulting access was (1) a write, and (2) to a
	// copy-on-write page.  If not, panic.
	// Hint:
	//   Use the read-only page table mappings at uvpt
	//   (see <inc/memlayout.h>).

	// LAB 4: Your code here.
	// verify write operation and COW bit
	if(!(err&FEC_WR)||!(uvpt[((uint32_t)addr)>>PGSHIFT]&PTE_COW))
		panic("error");

	// Allocate a new page, map it at a temporary location (PFTEMP),
	// copy the data from the old page to the new page, then move the new
	// page to the old page's address.
	// Hint:
	//   You should make three system calls.

	// LAB 4: Your code here.
	tmp=(void*)PFTEMP;
	origin=(void*)ROUNDDOWN((uint32_t)addr,PGSIZE);

	// allocate a page onto tmp region to accommodate copied page
	if(sys_page_alloc(0,tmp,PTE_W)<0)
		panic("error");
	// copy original page to tmp region
	memmove(tmp,origin,PGSIZE);
	// map original va to newly copyied page
	if(sys_page_map(0,tmp,0,origin,PTE_W)<0)
		panic("error");
	// unmap tmp, finish copy-on-write
	if(sys_page_unmap(0,tmp)<0)
		panic("error");
}
```
这部分比较简单，总的来说就是检查是不是copy-on-write，是的话分配给临时区一个页面然后拷贝，接着原先区映射到临时区映射的页，最后临时区取消映射

接下来是duppage：
```
//
// Map our virtual page pn (address pn*PGSIZE) into the target envid
// at the same virtual address.  If the page is writable or copy-on-write,
// the new mapping must be created copy-on-write, and then our mapping must be
// marked copy-on-write as well.  (Exercise: Why do we need to mark ours
// copy-on-write again if it was already copy-on-write at the beginning of
// this function?)
//
// Returns: 0 on success, < 0 on error.
// It is also OK to panic on error.
//
static int
duppage(envid_t envid, unsigned pn)
{
	// LAB 4: Your code here.
	extern volatile pte_t uvpt[];
	void* addr=(void*)(pn*PGSIZE);
	pte_t pte=uvpt[pn];

	// duppage only if the page is present and belongs to user
	if((pte&PTE_P)&&(pte&PTE_U)){
		if((pte&PTE_W)||(pte&PTE_COW)){
			if(sys_page_map(0,addr,envid,addr,PTE_COW)<0)
				panic("error");
			if(sys_page_map(0,addr,0,addr,PTE_COW)<0)
				panic("error");
		}else{
			if(sys_page_map(0,addr,envid,addr,0)<0)
				panic("error");
		}
	}	
	
	return 0;
}
```
这里要理解的两个问题是：
1. 为什么要先设置子进程为cow，再设置自己为cow
2. 为什么在函数刚进入时我们的这个页面假如已经cow了，为什么后面还要再设置为cow？

对1，答案比较清晰，如果先设置自己的为cow，那么有可能在自己的设置完到设置子进程的之前这段间隙，触发cow，那么子进程的cow就会指向父进程新的这个页面，可父进程对这个页面的权限确已经不再是cow了，那么子进程运行时，只要一直不写这个页面，就一直会使用这个页面，但父进程确可能会随时写入（不会触发cow了），在子进程看来，就是有人一直在篡改自己的只读数据。

对2，其实与1情况类似，但让我却想了很久一会儿。假如我们只对writeable的页面子、父进程都设置cow，对cow的页面只设置子进程cow，那么假如在判断完writeable or cow并判定为cow后，到设置子页面cow前父进程对这个页面触发了cow，那么仍然会回到1中的情况。因此，我们必须重新设置一次。

总的来说，也就是确保子进程对一个页标记为cow后，父进程必须重新对这个页标记一次cow，确保不出现子进程标为cow，父进程却标为writeable的情况。

最后我们来看fork
```
//
// User-level fork with copy-on-write.
// Set up our page fault handler appropriately.
// Create a child.
// Copy our address space and page fault handler setup to the child.
// Then mark the child as runnable and return.
//
// Returns: child's envid to the parent, 0 to the child, < 0 on error.
// It is also OK to panic on error.
//
// Hint:
//   Use uvpd, uvpt, and duppage.
//   Remember to fix "thisenv" in the child process.
//   Neither user exception stack should ever be marked copy-on-write,
//   so you must allocate a new page for the child's user exception stack.
//
envid_t
fork(void)
{
	// LAB 4: Your code here.
	extern unsigned char end[];
	envid_t childid;
	uint32_t addr;
	
	// set handler every time
	// to ensure it non-null
	set_pgfault_handler(pgfault);

	// perform fork
	if((childid=sys_exofork())<0)
		return -1;

	// child process
	if(childid==0){
		// update thisenv global variable
		thisenv=&envs[ENVX(sys_getenvid())];

		return 0;
	}

	// this following is parent process

	// it's necessary to duplicate stack page first
	// cuz that's the most frequently written page
	duppage(childid,ROUNDDOWN((uint32_t)&addr,PGSIZE)/PGSIZE);

	// allocate exception stack for child process
	// and copy to it with PTE_W directly set
	if(sys_page_alloc(0,(void*)PFTEMP,PTE_W)<0)
		panic("Fail to allocate page");
	memmove((void*)PFTEMP,(void*)(UXSTACKTOP-PGSIZE),PGSIZE);
	if(sys_page_map(0,(void*)PFTEMP,childid,(void*)(UXSTACKTOP-PGSIZE),PTE_W)<0)
		panic("Fail to map page");
	if(sys_page_unmap(0,(void*)PFTEMP)<0)
		panic("Fail to unmap page");

	// duplicate all other pages
	for(addr=0;addr<(uint32_t)end;addr+=PGSIZE)
		duppage(childid,addr/PGSIZE);
	
	// mark child process as runnable
	sys_env_set_status(childid,ENV_RUNNABLE);
	
	return childid;
}
```
比较重要的是，一定要先dup stack page（心痛，debug好久），同时exception stack直接分配而不是cow，其余的就比较简单了

但至此，还有一些兼容代码需要编写，首先是我个人的一个bug：
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
		if((*va_pte)&PTE_P)
			page_remove(pgdir,va);
		pp->pp_ref++;
	}
	
	// set up page table entry
	*va_pte=(page2pa(pp)|perm|PTE_P);
	// flush tlb is always necessary
	tlb_invalidate(pgdir,va);
	
	return 0;
}
```
之前tlb_invalidate写在了if里面：
```
	if(page2pa(pp)!=PTE_ADDR(*va_pte)){
		if((*va_pte)&PTE_P){
			page_remove(pgdir,va);
        	tlb_invalidate(pgdir,va);
        }
		pp->pp_ref++;
	}
```
但实际上只要pte被更新就应当刷新tlb

另外，如果是使用sysenter/sysexit实现系统调用，这里需要追加对ebp的保存：
```
static inline 
void save_curenv_trapframe(int syscallno){
	asm volatile("mov %%esi,%0":"=a"(curenv->env_tf.tf_eip));
	asm volatile("mov (%%ebp),%0":"=a"(curenv->env_tf.tf_esp));
	asm volatile("mov (%1),%0":"=a"(curenv->env_tf.tf_regs.reg_ebp):"a"(curenv->env_tf.tf_esp));
	asm volatile("mov 0xc(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_edx));
	asm volatile("mov 0x10(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_ecx));
	asm volatile("mov 0x14(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_ebx));
	asm volatile("mov 0x18(%%ebp),%0":"=a"(curenv->env_tf.tf_regs.reg_edi));
	asm volatile("mov %%es,%0":"=a"(curenv->env_tf.tf_es));
	asm volatile("mov %%ds,%0":"=a"(curenv->env_tf.tf_ds));
}
```
需要解引用两次

至此，copy-on-write算是完成了：
```
[00000000] new env 00001000
1000: I am ''
[00001000] new env 00001001
[00001000] new env 00001002
[00001000] exiting gracefully
[00001000] free env 00001000
1001: I am '0'
[00001001] new env 00002000
[00001001] new env 00001003
[00001001] exiting gracefully
[00001001] free env 00001001
2000: I am '00'
[00002000] new env 00002001
[00002000] new env 00001004
[00002000] exiting gracefully
[00002000] free env 00002000
2001: I am '000'
[00002001] exiting gracefully
[00002001] free env 00002001
1002: I am '1'
[00001002] new env 00003001
[00001002] new env 00003000
[00001002] exiting gracefully
[00001002] free env 00001002
3000: I am '11'
[00003000] new env 00002002
[00003000] new env 00001005
[00003000] exiting gracefully
[00003000] free env 00003000
3001: I am '10'
[00003001] new env 00004000
[00003001] new env 00001006
[00003001] exiting gracefully
[00003001] free env 00003001
4000: I am '100'
[00004000] exiting gracefully
[00004000] free env 00004000
2002: I am '110'
[00002002] exiting gracefully
[00002002] free env 00002002
1003: I am '01'
[00001003] new env 00003002
[00001003] new env 00005000
[00001003] exiting gracefully
[00001003] free env 00001003
5000: I am '011'
[00005000] exiting gracefully
[00005000] free env 00005000
3002: I am '010'
[00003002] exiting gracefully
[00003002] free env 00003002
1004: I am '001'
[00001004] exiting gracefully
[00001004] free env 00001004
1005: I am '111'
[00001005] exiting gracefully
[00001005] free env 00001005
1006: I am '101'
[00001006] exiting gracefully
[00001006] free env 00001006
```