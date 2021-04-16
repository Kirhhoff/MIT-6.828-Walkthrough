# Challenge 7: Atomic Fork
> **Challenge! Your implementation of fork makes a huge number of system calls. On the x86, switching into the kernel using interrupts has non-trivial cost. Augment the system call interface so that it is possible to send a batch of system calls at once. Then change fork to use this interface.**
> 
> How much faster is your new fork?
> 
> You can answer this (roughly) by using analytical arguments to estimate how much of an improvement batching system calls will make to the performance of your fork: How expensive is an int 0x30 instruction? How many times do you execute int 0x30 in your fork? Is accessing the TSS stack switch also expensive? And so on...
> 
> Alternatively, you can boot your kernel on real hardware and really benchmark your code. See the RDTSC (read time-stamp counter) instruction, defined in the IA32 manual, which counts the number of clock cycles that have elapsed since the last processor reset. QEMU doesn't emulate this instruction faithfully (it can either count the number of virtual instructions executed or use the host TSC, neither of which reflects the number of cycles a real CPU would require).

这里没有对性能差别跑benchmark啥的，只是把fork的操作正好到一次系统调用中完成了而已。相比之下，在一次系统调用中完成fork比先前的分多次fork有很多好处：
1. 代码简洁
2. 不需要考虑栈覆盖的问题（因此sysfork不需要非得写成内联函数了）
3. 设置cow页面标记的时候不需要考虑父/子进程顺序以及二次设置父进程cow的问题了
4. 减少系统调用次数
5. 许多操作得以完全在内核态进行，这样就可以省去许多不必要的安全检查（毕竟我们默认内核本身的行为是完全安全的），而且可以复用很多数据访问（不需要那么多次envid2env转换之类的了）

可以看出，真是好处多多啊！

总的来说，这里把之前的用户态duppage，以及sys_page_map等搬到内核态进行了重写，把可以优化的地方都优化了一下。而且这里的所有代码都是无锁的，锁的控制都放在了最外层的sys_fork里了。

总体上来看，调用处变为了如下形式：
```
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
	if((childid=sys_fork(end))<0)
		return -1;

	// child process
	if(childid==0){
		// update thisenv global variable
		thisenv=&envs[ENVX(sys_getenvid())];

		return 0;
	}

	return childid;
}
```
父进程fork结束后基本什么都不需要做，返回即可，子进程更新一下thisenv。这里的sys_fork传入了一个end参数，主要是因为目前内核态的env没有track这个进程用到了哪些虚拟内存地址，因此这个信息需要由用户提供给内核用以设置页面映射。

好了，sysfork的总体代码如下：
```
static envid_t
sys_fork(unsigned char end[])
{
	uint32_t addr;
	int err;
	struct Env* childenv;

	lock_env();
	lock_page();

	// create env for child process
	if((childenv=fork_exofork())==0){
		unlock_page();
		unlock_env();
		return -E_NO_MEM;
	}
	
	// duplicate stack page
	fork_duppage(childenv,(void*)ROUNDDOWN(curenv->env_tf.tf_esp,PGSIZE));
	
	// allocate exception stack for child process
	// and copy to it with PTE_W directly set

	if((err=fork_page_alloc(curenv,(void*)PFTEMP,PTE_W))<0){
		unlock_page();
		unlock_env();
		return err;
	}

	memmove((void*)PFTEMP,(void*)(UXSTACKTOP-PGSIZE),PGSIZE);

	if((err=fork_page_map(childenv,(void*)PFTEMP,(void*)(UXSTACKTOP-PGSIZE),PTE_W))<0){
		unlock_page();
		unlock_env();	
		return err;
	}

	page_remove(curenv->env_pgdir,(void*)PFTEMP);

	// duplicate all other pages
	for(addr=0;addr<(uint32_t)end;addr+=PGSIZE)
		fork_duppage(childenv,(void*)addr);
	
	// mark child process as runnable
	childenv->env_status=ENV_RUNNABLE;

	unlock_page();
	unlock_env();

	return childenv->env_id;
}
```
这里的话，用fork前缀来标识所有被“魔改”后的函数版本。锁的话基本就是把整个sys_fork作为一个临界区来对待了。这里来逐一对比这些修改了的方法。

**sys_exofork -> fork_exofork**
```
static struct Env*
fork_exofork(void)
{
	struct Env* e;

	// allocate empty env
	if(env_alloc(&e,curenv->env_id)<0)
		return 0;

	// child process has the same register states
	// except for %eax, which implies the return value,
	// 0 for child process
	e->env_tf=curenv->env_tf;
	e->env_tf.tf_regs.reg_eax=0;

	// inherit parent's page fault handler
	e->env_pgfault_upcall=curenv->env_pgfault_upcall;

	// set up kernel virtual memeory
	if(setupkvm(e->env_pgdir)<0){
		env_free(e);
		return 0;
	}
	
	return e;
}

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

	lock_env();

	lock_page();
	// allocate empty env
	if(env_alloc(&e,curenv->env_id)<0){
		unlock_page();
		unlock_env();
		return -E_NO_FREE_ENV;
	}
	unlock_page();


	// mark child process as not runnable for the moment
	e->env_status=ENV_NOT_RUNNABLE;

	// child process has the same register states
	// except for %eax, which implies the return value,
	// 0 for child process
	e->env_tf=curenv->env_tf;
	e->env_tf.tf_regs.reg_eax=0;

	// inherit parent's page fault handler
	e->env_pgfault_upcall=curenv->env_pgfault_upcall;

	unlock_env();

	lock_page();	
	// set up kernel virtual memeory
	if(setupkvm(e->env_pgdir)<0){
		unlock_page();
		return -E_NO_MEM;
	}
	unlock_page();
	return e->env_id;
}
```
这里修改的不多，主要在以下几个方面：
1. 返回env而不是envid。在整个过程中基本都用不到envid了，实际都是拿到envid转化成env
2. 删去了用来同步的锁
3. 不再设置ENV_NOT_RUNNABLE这个中间状态了

怎么样，是不是清爽了很多~

**duppage -> fork_duppage**
```
static int
fork_duppage(struct Env* childenv, void* addr)
{
	// LAB 4: Your code here.
	pte_t pte=*pgdir_walk(curenv->env_pgdir,addr,0);
	int err;

	// duppage only if the page is present and belongs to user
	if((pte&PTE_P)&&(pte&PTE_U)){
		if((pte&PTE_W)||(pte&PTE_COW)){
			if((err=fork_page_map(childenv,addr,addr,PTE_COW))<0)
				return err;
			if((pte&PTE_W)&&!(pte&PTE_COW)){
				if((err=fork_page_map(curenv,addr,addr,PTE_COW))<0)
					return err;
			}
		}else{
			if((err=fork_page_map(childenv,addr,addr,0))<0)
				return err;
		}
	}	
	
	return 0;
}

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
改动主要是：
1. 不再使用uvpt。这里就不需要通过这个用户态映射来访问pte了，而且实际也没法用，因为每个进程的uvpt都是不同的，除非穿给内核，否则无法访问
2. 由1可知，也不需要pn的计算了
3. 参数由envid变为env*
4. 调用的函数全部变为修改后的版本，而且可以看出参数也变了，后面再说
5. 返回错误值而不是直接panic
6. 最大的改动，应该是copy映射的逻辑。这里，只有在父进程的page标记为writeable且COW位没有设置的时候才修改父进程的pte，因为这里不需要再考虑race condition了

这里总体上看起来变化不大

**sys_page_alloc -> fork_page_alloc**
```
static int
fork_page_alloc(struct Env* e, void *va, int perm)
{
	struct PageInfo* pp;
	int err;

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

static int
sys_page_alloc(envid_t envid, void *va, int perm)
{
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
	
	lock_page();

	// allocate page
	if((pp=page_alloc(ALLOC_ZERO))==0){
		unlock_page();
		return -E_NO_MEM;
	}
	
	// insert
	if((err=page_insert(e->env_pgdir,pp,va,perm|PTE_U|PTE_P))<0){
		// free allocated page before return with error
		page_free(pp);
		unlock_page();
		return err;
	}

	unlock_page();

	return 0;
}
```
这里应该算是变化比较大的：
1. envid -> env*
2. 既然有env*了就不需要envid2env了
3. 删去了同步代码
4. 删去了对va的范围检查。因为默认是内核调用，因此也默认为是安全的范围（不是来自用户输入，可以认为安全）

看起来也简单了很多！很多！

**sys_page_map -> fork_page_map**
```
static int
fork_page_map(struct Env* dstenv,void *srcva,void *dstva, int perm)
{
	struct PageInfo* pp;
	pte_t *pte;
	int err;
	
	// check va arrange legitimacy
	// and page-aligned
	if((uint32_t)srcva>=UTOP
		||(uint32_t)dstva>=UTOP
		||((uint32_t)srcva%PGSIZE)!=0
		||((uint32_t)dstva%PGSIZE)!=0)
		return -E_INVAL;
	
	// check src already has mapping for srcva
	if((pp=page_lookup(curenv->env_pgdir,srcva,&pte))==0)
		return -E_INVAL;
	
	// perform mapping
	if((err=page_insert(dstenv->env_pgdir,pp,dstva,perm|PTE_U|PTE_P))<0)
		return err;

	return 0;
}

static int
sys_page_map(envid_t srcenvid, void *srcva,
	     envid_t dstenvid, void *dstva, int perm)
{
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
	
	lock_page();

	// check src already has mapping for srcva
	if((pp=page_lookup(srce->env_pgdir,srcva,&pte))==0){
		unlock_page();
		return -E_INVAL;
	}
	
	// verify that src grant write
	// permission only when it is granted that
	if(((~*pte)&PTE_W)&&(perm&PTE_W)){
		unlock_page();
		return -E_INVAL;
	}
	
	// perform mapping
	if((errno=page_insert(dste->env_pgdir,pp,dstva,perm|PTE_U|PTE_P))<0){
		unlock_page();
		return errno;
	}

	unlock_page();
	return 0;
}
```
1. 参数、返回错误、删去envid2env、去掉同步
2. 删去权限检查。因为这里传入的权限都是内核设置的，默认为安全
3. 保留边界检查。因为这里传入的va可能是用户传入的，需要防止非法地址

最后，sys_page_unmap可以用一行代码来替代：
```
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

	lock_page();
	page_remove(e->env_pgdir,va);
	unlock_page();

	return 0;	
}
```
这里也因为va是内核给的，所以不用检查，最后就精简成一行了：
```
	page_remove(curenv->env_pgdir,(void*)PFTEMP);
```

至此，对sys_fork的实现就可以正常工作了