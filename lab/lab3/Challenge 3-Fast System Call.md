# Challenge 3: Faset System Call
> **Implement system calls using the sysenter and sysexit instructions instead of using int 0x30 and iret.**
>
> **The sysenter/sysexit instructions were designed by Intel to be faster than int/iret. They do this by using registers instead of the stack and by making assumptions about how the segmentation registers are used. The exact details of these instructions can be found in Volume 2B of the Intel reference manuals.**
>
> **The easiest way to add support for these instructions in JOS is to add a sysenter_handler in kern/trapentry.S that saves enough information about the user environment to return to it, sets up the kernel environment, pushes the arguments to syscall() and calls syscall() directly. Once syscall() returns, set everything up for and execute the sysexit instruction. You will also need to add code to kern/init.c to set up the necessary model specific registers (MSRs). Section 6.1.2 in Volume 2 of the AMD Architecture Programmer's Manual and the reference on SYSENTER in Volume 2B of the Intel reference manuals give good descriptions of the relevant MSRs. You can find an implementation of wrmsr to add to inc/x86.h for writing to these MSRs here.**
>
> **Finally, lib/syscall.c must be changed to support making a system call with sysenter. Here is a possible register layout for the sysenter instruction:**

	eax                - syscall number
	edx, ecx, ebx, edi - arg1, arg2, arg3, arg4
	esi                - return pc
	ebp                - return esp
	esp                - trashed by sysenter
	
> **GCC's inline assembler will automatically save registers that you tell it to load values directly into. Don't forget to either save (push) and restore (pop) other registers that you clobber, or tell the inline assembler that you're clobbering them. The inline assembler doesn't support saving %ebp, so you will need to add code to save and restore it yourself. The return address can be put into %esi by using an instruction like leal after_sysenter_label, %%esi.**
>
> **Note that this only supports 4 arguments, so you will need to leave the old method of doing system calls around to support 5 argument system calls. Furthermore, because this fast path doesn't update the current environment's trap frame, it won't be suitable for some of the system calls we add in later labs.**
>
> **You may have to revisit your code once we enable asynchronous interrupts in the next lab. Specifically, you'll need to enable interrupts when returning to the user process, which sysexit doesn't do for you.**

> ps：不得不说，这个challenge做起来比之前的buddy system要头痛很多，因为之前的可以单点调试，而这里目前我们运行在用户空间，无法单点调试，内核的问题只能根据可怜的trapframe信息和一条条反汇编追踪，但好歹还是搞定了>>

总的来说，这个challenge的内容是为系统调用专门设置一个指令以及控制转移路径，以此提高系统调用的性能，用来类比int/iret的指令就是这里提到的sysenter/sysexit

我们首先回想一下，使用int $30时系统调用的流程：
- 将sysno以及调用的参数放在eax等寄存器中
- 到达int $30指令，触发中断（切换到内核态）
  - 到达handler，按照栈帧格式压栈中断号
  - 跳转到_alltraps，继续保存用户态信息，调用trap
    - 安全检查
    - 进入trapdispath分发
      - 根据trapno分发到syscall并将trapframe的内容取出作为参数传入
        - 进入syscall，根据sysno执行相应系统调用后返回
      - pop出trapframe（切换回用户态）

而采用新的sysenter/sysexit指令，可以省去许多不必要的安全检查与转发，过程大致如下（后面会详细介绍）：
- 将sysno以及调用的参数、**返回地址**等放在eax等寄存器中
- 到达sysenter指令，跳转到新的handler（切换到内核态）
  - 将寄存器中存储的参数压栈，直接调用syscall
    - 进入syscall，根据sysno执行相应系统调用后返回
  - 准备返回地址，执行sysexit指令（切换回用户态）

可以很明显看到，少了很多不必要的开销，毕竟通用往往意味着性能的下降，而对系统调用我们有许多convention（如必然要返回、确定要调用syscall）可以借来优化性能

OK，下面我们来看看这具体是怎么做到的，首先我们看看intel关于这两个指令的介绍：
![](https://img-blog.csdnimg.cn/2020030221065351.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/20200302210743486.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/20200302210810405.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/20200302210901919.PNG)

可以得知：
1. 跳转地址与返回地址都是通过特定通用寄存器以及MSR寄存器存储的
2. sysenter只能用于特权级3到0的转换，sysexit只能用于特权级0到3的转换
3. sysenter跳转到的地址(handler)是固定的，因此只要初始化一次即可
4. sysexit跳转的是用户地址，需要在执行sysexit前设置

关于设置MSR寄存器：                                                    
![](https://img-blog.csdnimg.cn/20200302211305923.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
这条指令只有在内核态才能执行，因此我们要在初始化阶段就设置好MSR

秉着这一想法，我们先来定义一下handler:
```
 .global sysenter_handler
 sysenter_handler:

	# prepare arguments for syscall
	pushl $0x0 # only four parameters supported, 0x0 for the fifth
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
	sysexit
```
基本就是准备参数、调用syscall，返回后准备用户信息然后sysexit

有了这个，我们就可以初始化MSR了，首先定义一下宏：
```
#define IA32_SYSENTER_CS (0x174)
#define IA32_SYSENTER_EIP (0x176)
#define IA32_SYSENTER_ESP (0x175)

#define wrmsr(msr,dx_val,ax_val) \
	asm volatile\
	("wrmsr"::"c"(msr),"d"(dx_val),"a"(ax_val));
```
而后初始化方法：
```
static void msr_init(){
	extern void sysenter_handler();
	uint32_t cs;
	asm volatile("movl %%cs,%0":"=r"(cs));
	wrmsr(IA32_SYSENTER_CS,0x0,cs)
	wrmsr(IA32_SYSENTER_EIP,0x0,sysenter_handler)
	wrmsr(IA32_SYSENTER_ESP,0x0,KSTACKTOP);
}
```
把它加在i386_init中就可以了

还剩最后一步，就是实现用户态的syscall函数了，声明和实现如下：
```
static int32_t
syscall(int num, int check, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
    __attribute__((noinline));
```
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

	// save user space %esp in %ebp passed into sysenter_handler
	asm volatile("pushl %ebp");
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
	
	// restore %ebp
	asm volatile("popl %ebp");
	
	if(check && ret > 0)
		panic("syscall %d returned %d (> 0)", num, ret);
	return ret;
}
```
注意这里要声明一下让编译器不要把syscall内联，因为我们这里内联汇编里定义了一个label名为.syslabel，如果内联这个函数的话会重复定义标号；同时这里也要先move参数再push ebp，因为static关键字使得对第三个之后的参数是通过栈访问的，而编译器并不会意识到ebp的值已经改变，会导致参数传递出错

看起来寥寥数语就说完了，但其中debug真的花了很久，有三方面，一方面是在一开始实现handler时误写成了pushl 0x0，就变成了访问内存0x0的值，而后压栈，心痛，花了很久才意识到这个问题，还是查了手册才想起来要$前缀来指示立即数，另一方面是标号的处理，用内联汇编写标号经常出错，而且很巧的是，保留
```
if(check && ret > 0)
    panic("syscall %d returned %d (> 0)", num, ret);
```
这段代码时标号就不会出错，而注释掉之后立马出错，反复追踪反汇编才意识到，这句话生成的汇编代码较长，加上这句话编译器就不会自动内联优化了，去掉这句话后syscall函数比较短，就自动内联优化了，当场去世，最后就是参数访问，要后push ebp才行，花了很久来调整代码

但是，也真的挺有成就感，终于搞定了>>
