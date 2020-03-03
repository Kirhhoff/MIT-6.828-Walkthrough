# Part B: Exception Handling, Page Faults, Breakpoints Exceptions, and System Calls

> 呼，这部分东西弄了很久，一方面是线上开学后，许多麻烦事要处理，另一方面是迁移了下工作环境，把原来虚拟机上的Ubuntu换到了wsl上（性能开销还都好说，但是渲染有点差，最重要的是对罗技鼠标的支持太差，滚轮迟钝得想杀人>>>而我鼠标又都是罗技，忍不下去了，后来正好看到wsl，就想试一下，但是wsl端口与windows共用，而我的windows版本对应的wsl端口有bug，开一段时间服务会把端口全部占满，网都上不了，make grade也没法正常运行，后来不得已又从wsl迁移到wsl2，为此还加入了Windows insider计划，结果英伟达显卡驱动又不适配了...说多都是泪，还好最后一切都work了，体验也比VMWare好了很多，总的环境迁移足足花了两天半，吐血>>），而且这一块儿又走入了一些误区，好待最后还是搞定了
>
> 最后想想还是把这些全部放在Part B中吧，因为它们之间关联更大一些，更连贯一些

OK，这一部分我们要实现什么呢？总的来说是这样，我们已经解决了内核初始化的部分，也解决了第一个进程的创建的问题，后面就是为用户空间提供系统调用以及错误检测、保护等等（也因此先解决中断、异常的functionality），而在这里，我们也会看到用户态与内核态的切换以及相关的信息是如何保存、恢复的

首先要问三个问题：
1. **中断/异常来临时，用户空间是怎么知道内核栈段的描述符和栈指针的？**
    这一部分的工作对我们是不可见的，是处理器自动进行的，首先，cs和ip都是存在中断描述符中的，这可以轻松得到，而内核的ss和sp指针到哪里去获取呢？答案是TSS段。因此每个cpu都需要一个TSS段来存放其全局内核栈段和栈指针
2. **用户空间的信息保存在哪里呢？**
    保存在内核栈上，也就是刚刚从TSS中切换完的那个。这里值得注意的是，我们的pgdir还是用的这个进程的，因为每个进程的pgdir都有一份内核空间的map
3. **总体过程是怎样的呢？**
   1. 中断/异常来临
   2. 从TSS中切换到内核栈，将当前用户cs，ip压栈
   3. 根据中断号（处理器自动产生）找到中断向量描述符，加载cs，ip
   4. 如果有error信息，push，没有则push 0保持栈结构一致
   5. push 中断号
   6. push es,ds 
   7. 将用户当前上下文（通用寄存器等等）压栈
   8. es，ds切换到内核数据段（手动进行）
   9. 栈指针压栈(即Trapframe开始地址，也就是&Trapframe)，作为参数调用C语言trap函数，根据异常号处理异常
   10. 如果要返回到用户程序，pop trapframe的内容即可恢复用户程序（这里处理器会自动从内核态再切换到用户态）


这样，我们来看看ex4:
> Edit trapentry.S and trap.c and implement the features described above. The macros TRAPHANDLER and TRAPHANDLER_NOEC in trapentry.S should help you, as well as the T_* defines in inc/trap.h. You will need to add an entry point in trapentry.S (using those macros) for each trap defined in inc/trap.h, and you'll have to provide _alltraps which the TRAPHANDLER macros refer to. You will also need to modify trap_init() to initialize the idt to point to each of these entry points defined in trapentry.S; the SETGATE macro will be helpful here.
>
> Your _alltraps should:
>
> 1. push values to make the stack look like a struct Trapframe
> 2. load GD_KD into %ds and %es
> 3. pushl %esp to pass a pointer to the Trapframe as an argument to trap()
> 4. call trap (can trap ever return?)
> Consider using the pushal instruction; it fits nicely with the layout of the struct Trapframe.
> 
> Test your trap handling code using some of the test programs in the user directory that cause exceptions before making any system calls, such as user/divzero. You should be able to get make grade to succeed on the divzero, softint, and badsegment tests at this point.

那么根据提示，_alltraps的内容如下：
```
_alltraps:

	# save %ds and %es
	# later we will switch to
	# kernel data selector
	pushl %ds
	pushl %es
	
	# save general registers
	# also keep Trapframe structure
	pushal
	
	# switch to kernel
	# data selector
	movl $GD_KD,%eax
	movl %eax,%es
	movl %eax,%ds

	# pass parameter
	# Trapframe* tf to trap
	pushl %esp
	
	call trap
```
这里一个一开始让我蛋疼了很久的问题是：直到_all_traps到底哪些信息已经被保留了？或者说，一开始处理器到底自动保存了哪些信息？后来又仔细看了一下，才找到：
> Let's put these pieces together and trace through an example. Let's say the processor is executing code in a user environment and encounters a divide instruction that attempts to divide by zero.
>
> 1. The processor switches to the stack defined by the SS0 and ESP0 fields of the TSS, which in JOS will hold the values GD_KD and KSTACKTOP, respectively.
> 2. The processor pushes the exception parameters on the kernel stack, starting at address KSTACKTOP:
> 
>                     +--------------------+ KSTACKTOP             
>                     | 0x00000 | old SS   |     " - 4
>                     |      old ESP       |     " - 8
>                     |     old EFLAGS     |     " - 12
>                     | 0x00000 | old CS   |     " - 16
>                     |      old EIP       |     " - 20 <---- ESP 
>                     +--------------------+             
>	
> 3. Because we're handling a divide error, which is interrupt vector 0 on the x86, the processor reads IDT entry 0 and sets CS:EIP to point to the handler function described by the entry.
> 4. The handler function takes control and handles the exception, for example by terminating the user environment.

注意3的末尾设置了CS:EIP，也就是说这一瞬间跳转到了我们宏定义的handler，因此一切就明了了

而对handler的声明我一开始也有点懵，主要是不知道哪些有error code，reference 1的那个手册语焉不详，后来看了一下reference 2 intel的手册，章节开头给出一张表才明确了：
![](https://img-blog.csdnimg.cn/20200301183213387.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/20200301183242960.PNG)
因此handler定义如下：
```
#define INT_DEFINE(name) \
	void name();
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
TRAPHANDLER_NOEC(_syscall,48)
```
初始化部分：
```
INT_DEFINE(divide)
INT_DEFINE(debug)
INT_DEFINE(nmi)
INT_DEFINE(brkpt)
INT_DEFINE(pflow)
INT_DEFINE(_bound)
INT_DEFINE(illop)
INT_DEFINE(device)
INT_DEFINE(dblflt)
INT_DEFINE(coproc)
INT_DEFINE(tss)
INT_DEFINE(segnp)
INT_DEFINE(stack)
INT_DEFINE(gpflt)
INT_DEFINE(pgflt)
INT_DEFINE(_res)
INT_DEFINE(fperr)
INT_DEFINE(_align)
INT_DEFINE(mchk)
INT_DEFINE(simderr)
```
init方法部分：
```
SETGATE(idt[0],STS_TG32,GD_KT,divide,0);
SETGATE(idt[1],STS_TG32,GD_KT,debug,0);
SETGATE(idt[2],STS_TG32,GD_KT,nmi,0);
SETGATE(idt[3],STS_TG32,GD_KT,brkpt,0);
SETGATE(idt[4],STS_TG32,GD_KT,pflow,0);
SETGATE(idt[5],STS_TG32,GD_KT,_bound,0);
SETGATE(idt[6],STS_TG32,GD_KT,illop,0);
SETGATE(idt[7],STS_TG32,GD_KT,device,0);
SETGATE(idt[8],STS_TG32,GD_KT,dblflt,0);
SETGATE(idt[9],STS_TG32,GD_KT,coproc,0);
SETGATE(idt[10],STS_TG32,GD_KT,tss,0);
SETGATE(idt[11],STS_TG32,GD_KT,segnp,0);
SETGATE(idt[12],STS_TG32,GD_KT,stack,0);
SETGATE(idt[13],STS_TG32,GD_KT,gpflt,0);
SETGATE(idt[14],STS_TG32,GD_KT,pgflt,0);
SETGATE(idt[15],STS_TG32,GD_KT,_res,0);
SETGATE(idt[16],STS_TG32,GD_KT,fperr,0);
SETGATE(idt[17],STS_TG32,GD_KT,_align,0);
SETGATE(idt[18],STS_TG32,GD_KT,mchk,0);
SETGATE(idt[19],STS_TG32,GD_KT,simderr,0);
```
### Questions

1. What is the purpose of having an individual handler function for each exception/interrupt? (i.e., if all exceptions/interrupts were delivered to the same handler, what feature that exists in the current implementation could not be provided?)

2. Did you have to do anything to make the user/softint program behave correctly? The grade script expects it to produce a general protection fault (trap 13), but softint's code says int $14. Why should this produce interrupt vector 13? What happens if the kernel actually allows softint's int $14 instruction to invoke the kernel's page fault handler (which is interrupt vector 14)?
### Answers
1. 个人认为主要是为了栈结构的统一，对栈结构的保持（也就是push 0x0）是在每个handler进入_alltraps前进行的（并且这个跟“每个中断都要有一个handler”根本不是一回事儿，因为可以看到，对于具体函数的分发是在trap_dispatch函数里根据传入的trapframe指针里的trapno通过一个switch语句进行的，到那时才是“每个中断都要一个对应的函数”）
2. 这里是因为我们对中断向量表初始化的时候，设置了int $14指令的权限为0，因此用户态无权执行这条指令，这里换成其他int $n都是一样的，都会出发general protection，因为目前我们所有的权限都设置为了0   
---
接下来解决页错误、断点异常和系统调用

这里对页错误的处理还是panic，只要在dispatch增加一项即可（其他可能会涉及页面置换等等），对breakpoint的处理也比较简单，如下：
```
switch (tf->tf_trapno)
{
case T_PGFLT:
    page_fault_handler(tf);
    break;

case T_BRKPT:
    monitor(tf);
    break;

default:
    break;
}
```
### Questions:
> 3. The break point test case will either generate a break point exception or a general protection fault depending on how you initialized the break point entry in the IDT (i.e., your call to SETGATE from trap_init). Why? How do you need to set it up in order to get the breakpoint exception to work as specified above and what incorrect setup would cause it to trigger a general protection fault?
> 4. What do you think is the point of these mechanisms, particularly in light of what the user/softint test program does?
### Answers:
3. 这里需要把int $3的权限修改一下：
    ```
	SETGATE(idt[T_BRKPT],STS_TG32,GD_KT,int_handlers[T_BRKPT],3);
    ```
    否则触发的就是general protection
4. 用于debug，设置断点

接下来就来到最有意思的部分了，系统调用。

系统调用的实现是使用了一个中断号48，系统调用号放在%eax中，之多五个参数放在%edx，%ecx，%ebx，%edi，和%esi中，返回值放在%eax中。

下面看看ex7:
> Add a handler in the kernel for interrupt vector T_SYSCALL. You will have to edit kern/trapentry.S and kern/trap.c's trap_init(). You also need to change trap_dispatch() to handle the system call interrupt by calling syscall() (defined in kern/syscall.c) with the appropriate arguments, and then arranging for the return value to be passed back to the user process in %eax. Finally, you need to implement syscall() in kern/syscall.c. Make sure syscall() returns -E_INVAL if the system call number is invalid. You should read and understand lib/syscall.c (especially the inline assembly routine) in order to confirm your understanding of the system call interface. Handle all the system calls listed in inc/syscall.h by invoking the corresponding kernel function for each call.
>
> Run the user/hello program under your kernel (make run-hello). It should print "hello, world" on the console and then cause a page fault in user mode. If this does not happen, it probably means your system call handler isn't quite right. You should also now be able to get make grade to succeed on the testbss test.

有别于其他中断，系统调用后要接着返回，而且我们要把返回值放在%eax中，因此这里dispatch函数与之前的改动稍有不同：
```
switch (tf->tf_trapno)
{
case T_PGFLT:
    page_fault_handler(tf);
    break;

case T_BRKPT:
    monitor(tf);
    break;

case T_SYSCALL:
    tf->tf_regs.reg_eax=syscall(
            tf->tf_regs.reg_eax,	// sysno
            tf->tf_regs.reg_edx,	// arg0
            tf->tf_regs.reg_ecx,	// arg1
            tf->tf_regs.reg_ebx,	// arg2 
            tf->tf_regs.reg_edi,	// arg3
            tf->tf_regs.reg_esi);	// arg4
    env_pop_tf(tf);
    break;

default:
    break;
}
```
env_pop_tf将控制权重重新移交给用户程序，syscall的内容也比较简单：
```
// Dispatches to the correct kernel function, passing the arguments.
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
	// Call the function corresponding to the 'syscallno' parameter.
	// Return any appropriate return value.
	// LAB 3: Your code here.
	switch (syscallno) {
	case SYS_cputs:	
		sys_cputs((char*)a1,a2);
		return 0;
	case SYS_cgetc: return sys_cgetc();
	case SYS_getenvid: return sys_getenvid();
	case SYS_env_destroy: return sys_env_destroy(a1);
	default:
		return -E_INVAL;
	}
}
```
这里我们来简单看一下用户程序hello：
```
void
umain(int argc, char **argv)
{
	cprintf("hello, world\n");
	cprintf("i am environment %08x\n", thisenv->env_id);
}
```
注意这里的cprintf并不是kernel的那个cprintf，而是改为了用户态的实现，使用sys_puts，而内核态的cprintf是使用outb和inb等等实现的

实验的最后是对用户态程序初始化添加一些代码，ex8:
> Add the required code to the user library, then boot your kernel. You should see user/hello print "hello, world" and then print "i am environment 00001000". user/hello then attempts to "exit" by calling sys_env_destroy() (see lib/libmain.c and lib/exit.c). Since the kernel currently only supports one user environment, it should report that it has destroyed the only environment and then drop into the kernel monitor. You should be able to get make grade to succeed on the hello test.

很简单，只需要初始化thisenv即可：
```
thisenv = &envs[ENVX(sys_getenvid())];
```
最后ex9，我们需要进行对系统调用参数的检查，具体来说就是对内存地址合法性的检查：
```
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
	// LAB 3: Your code here.
	uint32_t start=ROUNDDOWN((uint32_t)va,PGSIZE);
	uint32_t end=ROUNDUP((uint32_t)va+len,PGSIZE);
	pte_t* pte;
	for(uint32_t itrva=start;itrva<end;itrva+=PGSIZE){
		pte=pgdir_walk(env->env_pgdir,(void*)itrva,0);
		if(!pte||((~*pte)&(PTE_P|perm))){
			user_mem_check_addr=(itrva<=(uint32_t)va)?(uint32_t)va:itrva;
			return -E_FAULT;
		}
	}
	return 0;
}
```
同时，对于内核态的页错误（原则上不应当出现）直接panic，在page_fault_handler中加一句:
```
// Handle kernel-mode page faults.
// LAB 3: Your code here.
if(!(tf->tf_cs&0x3))
    panic("Kernel page fault\n");
```
以及对debuginfo_eip的参数检查：
```
// Make sure this memory is valid.
// Return -1 if it is not.  Hint: Call user_mem_check.
// LAB 3: Your code here.
if(user_mem_check(curenv,(void*)usd,sizeof(struct UserStabData),PTE_U)<0)
    return -1;

stabs = usd->stabs;
stab_end = usd->stab_end;
stabstr = usd->stabstr;
stabstr_end = usd->stabstr_end;

// Make sure the STABS and string table memory is valid.
// LAB 3: Your code here.
if(user_mem_check(curenv,(void*)stabs,(char*)stab_end-(char*)stabs,PTE_U)<0
    ||user_mem_check(curenv,(void*)stabstr,stabstr_end-stabstr,PTE_U)<0)
    return-1;
```
关于ex中提到的问题，我们看一下trace的显示信息：
```
lline 740 rline 740
depth 0: ebp 0xefffff20, retadr 0xf0100edd, args 0x1
       kern/monitor.c:388: monitor+271
lline 2187 rline 2187
depth 1: ebp 0xefffff90, retadr 0xf0103f50, args 0xf0216000
       kern/trap.c:218: trap+164
lline 2278 rline 2278
depth 2: ebp 0xefffffb0, retadr 0xf0104148, args 0xefffffbc 0x0 0x0 0xeebfdfd0 0xefffffdc 0x0
       kern/syscall.c:69: syscall+-5
lline 133 rline 133
depth 3: ebp 0xeebfdfd0, retadr 0x800073, args 0x0 0x0
       lib/libmain.c:26: libmain+53
lline 8 rline 8
depth 4: ebp 0xeebfdff0, retadr 0x800031, args
       lib/entry.S:34: <unknown>+-5
Incoming TRAP frame at 0xeffffe9c
kernel panic at kern/trap.c:296: Kernel page fault
```
可以看到trace到最后trace出来一个页错误，应该是trace的某一步访问了不该访问的地址（现在是在内核态），我们在页错误加个print_trapframe看到信息如下：
```
Incoming TRAP frame at 0xeffffe9c
TRAP frame at 0xeffffe9c
  edi  0x00000000
  esi  0xfffffffb
  ebp  0xefffff20
  oesp 0xeffffebc
  ebx  0x00000000
  edx  0x000003d5
  ecx  0x000003d4
  eax  0x00000000
  es   0x----0010
  ds   0x----0010
  trap 0x0000000e Page Fault
  cr2  0x00000000
  err  0x00000000 [kernel, read, not-present]
  eip  0xf010073b
  cs   0x----0008
  flag 0x00000006
kernel panic at kern/trap.c:297: Kernel page fault
```
看到cr2（触发页错误的虚拟地址）是0x0，就可以想到，应该是边界条件的问题：
```
while(true){
    int arg_index;
    struct Eipdebuginfo info;
    
    saved_ebp=*cur_ebp;
    ret_adr=*(cur_ebp+1);
    debuginfo_eip((uintptr_t)ret_adr,&info);
    cprintf("depth %d: ebp 0x%x, retadr 0x%x, args",depth,cur_ebp,ret_adr);
    
    for(arg_index=0;arg_index<info.eip_fn_narg;arg_index++)
        cprintf(" 0x%x",*(cur_ebp+2+arg_index));
    cprintf("\n       %s:%d: %.*s+%d\n",info.eip_file,info.eip_line,info.eip_fn_namelen,info.eip_fn_name,(uint32_t)ret_adr-(uint32_t)info.eip_fn_addr-5);
    
    /*
        0x0 is the base address of kernel stack.
        Current ebp reachs 0x0, which implies, we have reached the
        root of the calling nest.		
    */
    if((uint32_t)cur_ebp==(uint32_t)0x0)
        break;

    cur_ebp=(uint32_t*)saved_ebp;// Track back to the base address of caller.
    depth++;// Update the trace depth.
}
```
可以看到，当cur_ebp变为0时，先重新dereference了一次才判断退出的，而0x0是在用户地址空间内，并没有分配，也因此出现了non-present错误