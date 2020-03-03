# Challenge 1: Refactor
> **You probably have a lot of very similar code right now, between the lists of TRAPHANDLER in trapentry.S and their installations in trap.c. Clean this up. Change the macros in trapentry.S to automatically generate a table for trap.c to use. Note that you can switch between laying down code and data in the assembler by using the directives .text and .data.**

这个完全就是代码重构的问题了。首先我们要魔改一下TRAPHANDLER的宏定义，使其生成的汇编码长度相同，我们加一个.align指令：
```
#define TRAPHANDLER(name, num)						\
	.globl name;		/* define global symbol for 'name' */	\
	.type name, @function;	/* symbol type is function */		\
	.align 16;		/* align function definition */		\
	name:			/* function starts here */		\
	pushl $(num);							\
	jmp _alltraps

/* Use TRAPHANDLER_NOEC for traps where the CPU doesn't push an error code.
 * It pushes a 0 in place of the error code, so the trap frame has the same
 * format in either case.
 */
#define TRAPHANDLER_NOEC(name, num)					\
	.globl name;							\
	.type name, @function;						\
	.align 16;							\
	name:								\
	pushl $0;							\
	pushl $(num);							\
	jmp _alltraps
```
这里之所以align 16个字节，是因为TRAPHANDLER_NOEC生成的指令长度是9个字节，向上取2的整幂，就到16了。这样一来，每个handler的长度都是16字节了，我们在原来的宏定义前面加一个标号：
```
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
 TRAPHANDLER_NOEC(_syscall,48)
```
在init函数里面extern一下，就可以批量设定了：
```
extern long int_handlers[][4];
for(int i=0;i<20;i++)
    SETGATE(idt[i],STS_TG32,GD_KT,int_handlers[i],0);
SETGATE(idt[T_BRKPT],STS_TG32,GD_KT,int_handlers[T_BRKPT],3);

extern void _syscall();
SETGATE(idt[T_SYSCALL],STS_TG32,GD_KT,_syscall,3);
```
但是syscall没办法，因为它不是按index增长的