# Challenge 1: Fine-grained Locking
> **The big kernel lock is simple and easy to use. Nevertheless, it eliminates all concurrency in kernel mode. Most modern operating systems use different locks to protect different parts of their shared state, an approach called fine-grained locking. Fine-grained locking can increase performance significantly, but is more difficult to implement and error-prone. If you are brave enough, drop the big kernel lock and embrace concurrency in JOS!**
>
> **It is up to you to decide the locking granularity (the amount of data that a lock protects). As a hint, you may consider using spin locks to ensure exclusive access to these shared components in the JOS kernel:**
>
> - **The page allocator.**
> - **The console driver.**
> - **The scheduler.**
> - **The inter-process communication (IPC) state that you will implement in the part C.**

关于加锁，这里最终是加了三种锁来替代BKL：
1. env锁
2. page锁
3. console锁
4. ipc锁

其中3算是1的一种优化。并没有实现调度器锁，原因后面解释。

关于加锁的思路，一开始我以为很简单，但真正实现起来其实还是花了许多功夫的，直到最后思路真正清晰起来，也算游刃有余了。总体上来说，对任意一种锁，加锁时都要考虑两方面的问题：
1. 在哪里要加锁---临界区
2. 锁要加在哪里---DAG（Directed Acyclic Graph）

这里1和2是两个层次的问题，只有将1考虑清楚后，才能着手解决2。要判定临界区，就是要判断哪一部分代码整体必须是原子操作，也就知道在哪里要加锁。但是只解决1并不能开始加锁，因为存在锁的嵌套问题，也就是要避免在获取了锁的情况下再次获取锁的情况发生（这里并没有实现depth，一方面题目本身并没有提到，另一方面，笔者在前一段时间看了些linux内核代码，对自旋锁，linux中也没有使用depth，只在内核抢占计数上使用了depth，因此最终没有采用depth），因此分析调用层级就变得必要了，这里就引入了有向无环图来解决。

比较有趣的是，这四种锁都有不同的特点：env锁应该算是最麻烦的一种，因为要解决类似per-cpu variable的问题，而一开始本人并没有想清楚这个问题，也花了很多功夫；page锁相对来说更标准一些，完全是一个公共内存读写同步的问题，因此简单些；做完env锁和page锁后，console相对就非常简单了，只需要在有限的几个出入口同步一下即可；最后的ipc锁，其实算是对env锁的一种优化。

那么下面开始逐个介绍。

env锁
---
首先，我们来看一下env结构的访问情况。总的来说，env的成员分为两种：
1. 可以被任何一个cpu修改的
2. 只有作为local variable(thisenv)时才会被owner cpu修改的

后者不会造成contention condition，因为其他cpu不会访问，所以临界区代码应当是在访问1时出现的。那么env的哪些成员是属于1的呢？通过分析所有的访问代码，我们不难发现其实主要是
1. env_status 
2. ipc相关成员

除了这些再加上env_free_list等的公共变量，这就是会产生临界区的代码了。根据这一准则我们可以画出如下图：
![](https://img-blog.csdnimg.cn/20200417132403505.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)

全部临界区可以归类如下：
- 灰色：临界区出口，只有锁的释放
- 浅绿：无锁代码，但实际调用了灰色的临界区退出点代码
- 深绿：无锁代码，lock ignorant，不包含任何锁的获取与释放
- 浅蓝：发生控制权移交的临界区入口，只有锁的获取
- 橙色：包含完整临界区，具有完整的锁的获取与释放，不会发生控制权移交
- 深蓝：包含完整临界区，具有完整的锁的获取与释放，但可能会发生控制权移交，这时实际只有锁的获取，等价于浅蓝的类型

图中的箭头，A->B表示A被B调用。因此这实际上是个分析代码构造DAG的过程，只有在出度为0的方框（蓝色系和橙色）中才应当出现锁的获取，入度为0（灰色，深绿色不算，因为它们并不知道锁的存在）或包含完整临界区（深蓝色和橙色）的方框中才应当出现锁的释放。

env锁的代码因为有很多与page锁一起出现，因此我们先不放出来，等后面和page锁的一起放出来

page锁
---
page锁相对env锁有两方面的简化：
1. 不存在local variable，直接在所有的公共访问处加锁同步即可
2. 不涉及控制权移交，不会出现只有获取/释放锁其中一个的迷惑代码

因此直接在所有访问处加锁同步即可，分析代码可以得到类似这样的图：
![](https://img-blog.csdnimg.cn/20200417153919788.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
这里仍然是一个DAG，箭头含义与原来类似，只有一些小的变化，关于颜色和箭头：
- 橙色：包含完整临界区，具有完整的锁的获取与释放
- 蓝绿色：临界区的中间部分代码，代码中无锁出现，包含对其他临界区代码的调用，并且方法位于pmap.c外
- 绿色：临界区的中间部分代码，代码中无锁出现，包含对其他临界区代码的调用，并且方法位于pmap.c内
- 粉红色：临界区的中间部分代码，代码中无锁出现，且不包含对其他临界区代码的调用（也即，入度为0）并且方法位于pmap.c内
- 黑色关联线：与env锁相同
- 橙色关联线：除了表示调用关系外，A->B还说明，B在调用A周围进行了获取/释放锁操作，B算是A的众多“闭包”的其中之一

在这里我们把env锁和page锁的代码一起放出来：
```
int
envid2env(envid_t envid, struct Env **env_store, bool checkperm)
{
    ...

	e = &envs[ENVX(envid)];
	lock_env();
	if (e->env_status == ENV_FREE || e->env_id != envid) {
		*env_store = 0;
		unlock_env();
		return -E_BAD_ENV;
	}

	unlock_env();

	...
}
```
```
static void
load_icode(struct Env *e, uint8_t *binary)
{
	...

	lock_page();
	for(ph=ph_start;ph<ph_end;ph++)
		if(ph->p_type==ELF_PROG_LOAD)
			region_alloc(e,(void*)(ph->p_va),ph->p_memsz);
	unlock_page();
	
    ...
	
    lock_page();
	region_alloc(e,(void*)(USTACKTOP-PGSIZE),PGSIZE);
	unlock_page();
}
```
```
void
env_create(uint8_t *binary, enum EnvType type)
{
	...

	lock_page();
	// allocate a new env
	if((errno=env_alloc(&e,0))<0){
		unlock_page();
		panic("panic %e",errno);
	}
	unlock_page();

    ...
}
```
```
void
env_destroy(struct Env *e)
{
    ...
	
    lock_page();
	env_free(e);
	unlock_page();

	...
}
```
```
void
env_pop_tf(struct Trapframe *tf)
{
	// Record the CPU we are running on for user-space debugging
	curenv->env_cpunum = cpunum();
	unlock_env();

	...
}
```
```
void
mp_main(void)
{
	...

	lock_env();
	sched_yield();
}
```
```
void
lapic_init(void)
{
	...

	lock_page();
	lapic = mmio_map_region(lapicaddr, 4096);
	unlock_page();
	
	...
}
```
```
void
user_mem_assert(struct Env *env, const void *va, size_t len, int perm)
{
	lock_env();
	lock_page();
	if (user_mem_check(env, va, len, perm | PTE_U) < 0) {
		cprintf("[%08x] user_mem_check assertion failure for "
			"va %08x\n", env->env_id, user_mem_check_addr);
		unlock_page();
		env_destroy(env);	// may not return
	}else
		unlock_page();
	unlock_env();
}
```
```
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
	unlock_env();

    ...
}
```
```
static int
sys_env_destroy(envid_t envid)
{
	...

	lock_env();
	env_destroy(e);
	unlock_env();
	return 0;
}
```
```
static void
sys_yield(void)
{
	lock_env();
	sched_yield();
}
```
```
static envid_t
sys_exofork(void)
{
	...

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
```
static int
sys_env_set_status(envid_t envid, int status)
{
	...

	lock_env();
	e->env_status=status;
	unlock_env();
	return 0;	
}
```
```
static int
sys_page_alloc(envid_t envid, void *va, int perm)
{
	...
	
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
```
static int
sys_page_map(envid_t srcenvid, void *srcva,
	     envid_t dstenvid, void *dstva, int perm)
{
	...
	
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
```
static int
sys_page_unmap(envid_t envid, void *va)
{
	...

	lock_page();
	page_remove(e->env_pgdir,va);
	unlock_page();

	return 0;	
}
```
```
static int
sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
{
	...

	lock_env();
	// assert target env is receiving
	if(!tarenv->env_ipc_recving||tarenv->env_status!=ENV_NOT_RUNNABLE){
		unlock_env();
		return -E_IPC_NOT_RECV;
	}
	
	// whether receiver is receiving a page mapping
	// if so, perform a series of securiy check of 
	// srcva and finally insert it to target env's
	// page mapping
	if((uint32_t)tarenv->env_ipc_dstva<UTOP){
		lock_page();
		if(((uint32_t)srcva>=UTOP||(uint32_t)srcva%PGSIZE!=0)// verify srcva is legitimate and page-aligned
				||((pp=page_lookup(curenv->env_pgdir,srcva,&srcpte))<0)// verify srcva resides in curenv's page mapping
				||((perm|PTE_SYSCALL)!=PTE_SYSCALL)// check perm legitimacy
				||((~*srcpte&PTE_W)&&(perm&PTE_W))){// check write permission grant
			unlock_page();
			unlock_env();
			return -E_INVAL;
		}
		// perform page mapping insertion
		if(page_insert(tarenv->env_pgdir,pp,tarenv->env_ipc_dstva,perm)<0){
			unlock_page();
			unlock_env();
			return -E_NO_MEM;
		}
		unlock_page();
	}

	// record informations
	tarenv->env_ipc_from=curenv->env_id;
	tarenv->env_ipc_value=value;

	// reset target environment to runnable state
	tarenv->env_ipc_recving=false;
	tarenv->env_status=ENV_RUNNABLE;

	// set return value for receiver
	tarenv->env_tf.tf_regs.reg_eax=0;
	unlock_env();
	return 0;
}
```
```
static int
sys_ipc_recv(void *dstva)
{

	lock_env();

	// first check whether expect to
	// receive a page mapping
	if((uint32_t)dstva<UTOP){
		// if so, verify dstva page-aligned
		if((uint32_t)dstva%PGSIZE!=0){
			unlock_env();
			return -E_INVAL;
		}
		
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
```
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
	int ret;
	uint32_t eflags;

	// Garbage collect if current enviroment is a zombie
	lock_env();
	if (curenv->env_status == ENV_DYING) {
		lock_page();
		env_free(curenv);
		unlock_page();
		curenv = NULL;
		sched_yield();
	}
	unlock_env();
	
	...

}
```
```
static void
trap_dispatch(struct Trapframe *tf)
{
	...

	case T_BRKPT:
		monitor(tf);
		tf->tf_eip++;
		lock_env();
		env_pop_tf(tf);
		break;

	default:
		break;
	}

	// Handle spurious interrupts
	// The hardware sometimes raises these because of noise on the
	// IRQ line or other reasons. We don't care.
	if (tf->tf_trapno == IRQ_OFFSET + IRQ_SPURIOUS) {
		cprintf("Spurious interrupt on irq 7\n");
		print_trapframe(tf);
		return;
	}

	// Handle clock interrupts. Don't forget to acknowledge the
	// interrupt using lapic_eoi() before calling the scheduler!
	// LAB 4: Your code here.
	if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
		lapic_eoi();
		lock_env();
		sched_yield();
	}
	
	// Unexpected trap: The user process or the kernel has a bug.
	print_trapframe(tf);
	if (tf->tf_cs == GD_KT)
		panic("unhandled trap in kernel");
	else {
		lock_env();
		env_destroy(curenv);
		unlock_env();
		return;
	}
}
```
```
void
trap(struct Trapframe *tf)
{
	...

	if ((tf->tf_cs & 3) == 3) {
		// Trapped from user mode.
		// Acquire the big kernel lock before doing any
		// serious kernel work.
		// LAB 4: Your code here.
		assert(curenv);

		lock_env();

		// Garbage collect if current enviroment is a zombie
		if (curenv->env_status == ENV_DYING) {
			lock_page();
			env_free(curenv);
			unlock_page();
			curenv = NULL;
			sched_yield();
		}

		unlock_env();
		// Copy trap frame (which is currently on the stack)
		// into 'curenv->env_tf', so that running the environment
		// will restart at the trap point.
		curenv->env_tf = *tf;
		// The trapframe on the stack should be ignored from here on.
		tf = &curenv->env_tf;
	}

	// Record that tf is the last real trapframe so
	// print_trapframe can print some additional information.
	last_tf = tf;

	// Dispatch based on what type of trap occurred
	trap_dispatch(tf);

	// If we made it to this point, then no other environment was
	// scheduled, so we should return to the current environment
	// if doing so makes sense.
	lock_env();
	if (curenv && curenv->env_status == ENV_RUNNING)
		env_run(curenv);
	else
		sched_yield();
}
```
```
void
page_fault_handler(struct Trapframe *tf)
{
	...

	if(curenv->env_pgfault_upcall!=0){
		
        ...

		lock_env();
		env_run(curenv);		
	}
	
destroy:
	// Destroy the environment that caused the fault.
	cprintf("[%08x] user fault va %08x ip %08x\n",
		curenv->env_id, fault_va, tf->tf_eip);
	print_trapframe(tf);
	lock_env();
	env_destroy(curenv);
	unlock_env();
}
```

在这里穿插一下对锁的获取和释放顺序的说明。这里可以看出来，当需要同时获取env锁和page锁时总是先获取env锁，对应的，同时释放时总是先释放page锁。可以证明的是，如果某处获取顺序错误，就可能会引发死锁，这里就不多赘述了。之所以先获取env锁，主要是因为env锁会涉及控制转移，在控制转移时（env_pop_tf/sched_halt）不会涉及page锁，但控制转移前总是处于env锁的holding状态，因此可以认为env锁的生命周期总是长于其他锁，因此env锁应当处于外围。

console锁
---
这个相比原先就简单了很多，只在出口入口把把关即可：

in锁：
```
int
cons_getc(void)
{
	int c;

	lock_console();

	// poll for any pending input characters,
	// so that this function works even when interrupts are disabled
	// (e.g., when called from the kernel monitor).
	serial_intr();
	kbd_intr();

	// grab the next character from the input buffer.
	if (cons.rpos != cons.wpos) {
		c = cons.buf[cons.rpos++];
		if (cons.rpos == CONSBUFSIZE)
			cons.rpos = 0;
		
		unlock_console();
		return c;
	}
	unlock_console();
	return 0;
}
```
out锁：
```
int
vcprintf(const char *fmt, va_list ap)
{
	int cnt = 0;

	lock_console();
	vprintfmt((void*)putch, &cnt, fmt, ap);
	unlock_console();
	return cnt;
}
```
注意这里必须要在vcprintf处加锁而不是putch处，因为要完整地输出一个句子，而不是单个字符。

ipc锁
---
其实个人感觉这个锁不加也罢，因为对env的保护实际就包含了对ipc的保护，但是还是加一下吧，需要对ipc的几个系统调用进行小修改：
```
static int
sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
{
	...

	lock_ipc();
	// assert target env is receiving
	if(!tarenv->env_ipc_recving||tarenv->env_status!=ENV_NOT_RUNNABLE){
		unlock_ipc();
		return -E_IPC_NOT_RECV;
	}
	
	// whether receiver is receiving a page mapping
	// if so, perform a series of securiy check of 
	// srcva and finally insert it to target env's
	// page mapping
	if((uint32_t)tarenv->env_ipc_dstva<UTOP){
		lock_page();
		if(((uint32_t)srcva>=UTOP||(uint32_t)srcva%PGSIZE!=0)// verify srcva is legitimate and page-aligned
				||((pp=page_lookup(curenv->env_pgdir,srcva,&srcpte))<0)// verify srcva resides in curenv's page mapping
				||((perm|PTE_SYSCALL)!=PTE_SYSCALL)// check perm legitimacy
				||((~*srcpte&PTE_W)&&(perm&PTE_W))){// check write permission grant
			unlock_page();
			unlock_ipc();
			return -E_INVAL;
		}
		// perform page mapping insertion
		if(page_insert(tarenv->env_pgdir,pp,tarenv->env_ipc_dstva,perm)<0){
			unlock_page();
			unlock_ipc();
			return -E_NO_MEM;
		}
		unlock_page();
	}

	// record informations
	tarenv->env_ipc_from=curenv->env_id;
	tarenv->env_ipc_value=value;

	// reset target environment to runnable state
	tarenv->env_ipc_recving=false;
	tarenv->env_status=ENV_RUNNABLE;

	// set return value for receiver
	tarenv->env_tf.tf_regs.reg_eax=0;
	unlock_ipc();
	return 0;
}
```
```
static int
sys_ipc_recv(void *dstva)
{

	lock_env();
	lock_ipc();

	// first check whether expect to
	// receive a page mapping
	if((uint32_t)dstva<UTOP){
		// if so, verify dstva page-aligned
		if((uint32_t)dstva%PGSIZE!=0){
			unlock_ipc();
			unlock_env();
			return -E_INVAL;
		}
		
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
	unlock_ipc();

	sched_yield();

	// actually never reach here
	return 0;
}
```
注意这里在recv里同时锁上了env和ipc，因为中间对env_status进行了修改，这个虽然是ipc的行为，但是会与其他修改env_status的操作形成contention，还是需要用env锁进行保护。