> Exercise 3. Take a look at the lab tools guide, especially the section on GDB commands. Even if you're familiar with GDB, this includes some esoteric GDB commands that are useful for OS work.

> Set a breakpoint at address 0x7c00, which is where the boot sector will be loaded. Continue execution until that breakpoint. Trace through the code in boot/boot.S, using the source code and the disassembly file obj/boot/boot.asm to keep track of where you are. Also use the x/i command in GDB to disassemble sequences of instructions in the boot loader, and compare the original boot loader source code with both the disassembly in obj/boot/boot.asm and GDB.

> Trace into bootmain() in boot/main.c, and then into readsect(). Identify the exact assembly instructions that correspond to each of the statements in readsect(). Trace through the rest of readsect() and back out into bootmain(), and identify the begin and end of the for loop that reads the remaining sectors of the kernel from the disk. Find out what code will run when the loop is finished, set a breakpoint there, and continue to that breakpoint. Then step through the remainder of the boot loader.

# Answer for exercise
我们来一段段地看这些代码。首先是boot.S
```
#include <inc/mmu.h>
```
添加需要用到的宏如SEG_NULL
```
.set PROT_MODE_CSEG, 0x8         # kernel code segment selector
.set PROT_MODE_DSEG, 0x10        # kernel data segment selector
.set CR0_PE_ON,      0x1         # protected mode enable flag
```
预先定义常量，分别是内核代码段选择子和内核数据段选择子，以及稍后用于打开cr0 PE位开启保护模式的常量

关于段选择子中每个bit的意义可以参考《Understanding Linux Kernel》第二章或者《x86汇编语言：从实模式到保护模式》的第十一章

关于linux中段的情况，比如内核（用户）代码、数据段的设置，可以参考《Understanding Linux Kernel》的第二章
```
.globl start
start:
  .code16                     # Assemble for 16-bit mode
  cli                         # Disable interrupts
  cld                         # String operations increment
```
.global伪指令将start标号声明为对外部文件可见的，用于链接

.code16伪指令声明下面的代码运行于16位模式下

cli,cld关中断，设置重复拷贝时的拷贝方向，开始为开启保护模式做准备
```
  # Set up the important data segment registers (DS, ES, SS).
  xorw    %ax,%ax             # Segment number zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment
```
0初始化寄存器，没啥好说的
```
  # Enable A20:
  #   For backwards compatibility with the earliest PCs, physical
  #   address line 20 is tied low, so that addresses higher than
  #   1MB wrap around to zero by default.  This code undoes this.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
```
恢复第21条地址线A20的正常功能。详细内容可以参考，大体上来说是这样：8086只有20条地址线A0-A19，那么这时当地址达到0xfffff时再自增一下就会回到0x00000，当时有部分程序依赖这个特性（。。。）。但后来的80286地址线增长到24条，为了保持兼容性，就强制第21条地址线为0，那么此时这个特性就又得以正常工作。这里是为了移除这个强制，之所以这里访问了键盘驱动器是因为早期这个电路是附加在键盘驱动器上的，所以要手动设置之。

下面就开始要设置全局段描述符表global descriptor table了
```
  # Switch from real to protected mode, using a bootstrap GDT
  # and segment translation that makes virtual addresses 
  # identical to their physical addresses, so that the 
  # effective memory map does not change during the switch.
  lgdt    gdtdesc
```
这里出现了一个定义在段尾的常量gdtdesc，我们先来看一下
```
# Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULL				# null seg
  SEG(STA_X|STA_R, 0x0, 0xffffffff)	# code seg
  SEG(STA_W, 0x0, 0xffffffff)	        # data seg

gdtdesc:
  .word   0x17                            # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```
这就是初始化阶段临时的全局段描述符表了，可以看到，这个表是放在紧跟着这段汇编代码之后的，首先一个对其伪指令，然后打了个标号gdt，接着是三个全局段描述符，那么标号gdt的值就是全局段描述符表的起始地址了。这三个段描述符是怎样的呢？第一个使用SEG_NULL宏定义，根据标准要置空，因为要防止全0的值一下子实误地访问到这里，第二第三个就是之前定义的那两个段选择子对应的段描述符了，都使用了SEG宏以及bit operation来很方便地初始化。

最后这个gdtdesc是什么鬼？而且它还用在了lgdt指令后面。lgdt是加载gdtr的指令，但是它间接取址的，也就是，他会把gdtdesc最为地址，将这个地址开始的48个位加载到gdtr中。gdtr的格式是怎样的呢？低16位放gdt界限值，高32位放gdt基址，这就解释了这两行指令，先定义一个word(这里的word是16位的)放界限值（低地址），然后一个long word放起始地址，也就是gdt标号值。这样lgdt对gdtdesc作取值操作就正好可以把恰当的值加载进gdtr寄存器。

OK，既然我们已经设配置好了临时的全局描述符表以及相应寄存器，下面我们就可以打开保护模式了
```
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
```
打开保护模式后，通过一条long jump指令切换处理器到32位模式
```
  # Jump to next instruction, but in 32-bit code segment.
  # Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg

  .code32                     # Assemble for 32-bit mode
protcseg:
  # Set up the protected-mode data segment registers
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
  movw    %ax, %ss                # -> SS: Stack Segment
```
在开启保护模式后，内存地址访问时使用的段寄存器含义也要做相应的改变，这里对段寄存器进行初始化。可以看到，这几个数据相关的段寄存器全部被初始化为了内核的数据段选择子，关于为什么相同，可以参考《Understanding Linux Kernel》第二章，Linux内核所有数据相关的段都是公用一个，因此段选择子都是相同的。
```
  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call bootmain
  
  # If bootmain returns (it shouldn't), loop.
spin:
  jmp spin  
```
接下来设置了栈指针，栈指针被初始化到这段代码开始处，也即是0x7c00，bootloader代码从此处向上延申，栈向下延申，可以通过反汇编或实际调试来验证这一点
```
protcseg:
  # Set up the protected-mode data segment registers
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
    7c32:	66 b8 10 00          	mov    $0x10,%ax
  movw    %ax, %ds                # -> DS: Data Segment
    7c36:	8e d8                	mov    %eax,%ds
  movw    %ax, %es                # -> ES: Extra Segment
    7c38:	8e c0                	mov    %eax,%es
  movw    %ax, %fs                # -> FS
    7c3a:	8e e0                	mov    %eax,%fs
  movw    %ax, %gs                # -> GS
    7c3c:	8e e8                	mov    %eax,%gs
  movw    %ax, %ss                # -> SS: Stack Segment
    7c3e:	8e d0                	mov    %eax,%ss
  
  # Set up the stack pointer and call into C.
  movl    $start, %esp
    7c40:	bc 00 7c 00 00       	mov    $0x7c00,%esp
  call bootmain
    7c45:	e8 cb 00 00 00       	call   7d15 <bootmain>
```
![调试结果](https://img-blog.csdnimg.cn/20200218203751407.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)

下一步就是跳转到C语言代码处了，进入bootmain(另外从代码中可以看出，进入bootmain后不应当再返回，如果返回就自行空转,....)

---
总的来说bootmain做了以下的事情：
1. 将MBR后的前8个扇区，也就是内核elf文件的文件头，读入内存
2. 根据文件头，把程序的每个段依次加载到它们要求的物理地址
3. 跳转到内核的开始处，正式将控制权转交给内核
我们来一一看一看
```
#define SECTSIZE	512
#define ELFHDR		((struct Elf *) 0x10000) // scratch space
```
首先是定义扇区大小，以及要把elf文件头加载到哪里。这里可以看到，是强行指定加载到了low memory区域的一个位置，用完即覆盖，无所谓了，但是有必要说明的一点是，现在还没有诸如malloc之类的lib函数，因此无法进行严格的内存分配，因此就强制类型转换手动分配了。
```
	struct Proghdr *ph, *eph;

	// read 1st page off disk
	readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

	// is this a valid ELF?
	if (ELFHDR->e_magic != ELF_MAGIC)
		goto bad;
```
首行是定义局部变量方便后续使用。

接着通过readseg将八个扇区读入ELFHDR起始的内存区域，最后一个参数0是相对于第一个扇区后定义的，因此是0。读入之后，检查验证码，瞅瞅是否是有效的ELF，不是的话gg

下面开始加载程序的不同段，每个程序段都有一个程序头来描述，它们紧邻着存在，首先找到它们的起始和结束处
```
	// load each program segment (ignores ph flags)
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	eph = ph + ELFHDR->e_phnum;
```
ph是找到program headers起始处相对于elf header起始处的偏移再与起始处相加得到的，再通过指针运算，得到eph也就是end of program header，进行遍历加载即可
```
	for (; ph < eph; ph++)
		// p_pa is the load address of this segment (as well
		// as the physical address)
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```
到这里就比较清晰了，根据每个segment指定的p_pa（也就是指定的起始地址），其大小(p_memsz)以及它在硬盘上偏移的扇区号即可将它们依次读入

最后
```
	// call the entry point from the ELF header
	// note: does not return!
	((void (*)(void)) (ELFHDR->e_entry))();
```
将内核入口点强制转化为一个返回值void，参数void的函数指针而后调用即可正式将执行权移交给内核

可以看到，相对来说重要的工作（偏硬件一些的）其实都在boot.S中搞定了，main.c这一部分的C代码只是读取内核跳转，C语言毕竟方便些。

# Questions:
1. At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?
2. What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded?
3. Where is the first instruction of the kernel?
4. How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
# Answer
1. ljmp指令后；设置cr0寄存器；
2. bootloder最后的指令是初始化栈指针，kernel的第一条指令是movw $0x1234,0x472，是内核entry.S的第一条指令
3. 0x10000c，1M地址后刚开始处，这个可以通过做实验看出来（ps：一开始支这个地方我搞错了，做到后面才反应过来自己的理解有问题）
先看看这句函数指针调用的反汇编（boot.asm）:
    ```
    for (; ph < eph; ph++)
        7d66:	83 c4 0c             	add    $0xc,%esp
        7d69:	eb e6                	jmp    7d51 <bootmain+0x3c>
	((void (*)(void)) (ELFHDR->e_entry))();
        7d6b:	ff 15 18 00 01 00    	call   *0x10018
    ```
    注意这里，不是写的call 0x10000c而是call *0x10018，也就是说先对0x10018解引用，再把那里的值作为跳转地址，我们调试一下看看：
    ```
    (gdb) b *0x7d6b
    Breakpoint 1 at 0x7d6b
    (gdb) c
    Continuing.
    The target architecture is assumed to be i386
    => 0x7d6b:	call   *0x10018

    Breakpoint 1, 0x00007d6b in ?? ()
    (gdb) x/10x 0x10018
    0x10018:	0x0010000c	0x00000034	0x00015364	0x00000000
    0x10028:	0x00200034	0x00280003	0x000e000f	0x00000001
    0x10038:	0x00001000	0xf0100000
    (gdb) si 1
    => 0x10000c:	movw   $0x1234,0x472
    0x0010000c in ?? ()
    ```
    嗯，就是这样
4. elf文件头的大小是预先设定好的八个扇区，后续程序段个数、大小是根据elf头中的信息得到的



