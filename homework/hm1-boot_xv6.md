- 按照要求，首先要
  > Find the address of _start, the entry point of the kernel:
  ```
  $ nm kernel | grep _start
  8010a48c D _binary_entryother_start
  8010a460 D _binary_initcode_start
  0010000c T _start
  ```
  这里首先nm命令的含义是：
  ```
  lumin@ubuntu:~/Documents/MIT-6.828/lab/obj/kern$ nm -h
  Usage: nm [option(s)] [file(s)]
  List symbols in [file(s)] (a.out by default).
  ```
  列出binary file中的符号，grep选出_start。结合lab1我们知道，这个_start是bootloader结束后进入内核时的跳转地址，0x1000c，恰在1M地址开始处。
  ```
  lumin@ubuntu:~/Documents/MIT-6.828/lab/obj/kern$ nm kernel| grep _start
  0010000c T _start
  ```
  我们再来看看entry.S开头处吧
  ```
  .globl		_start
  _start = RELOC(entry)

  .globl entry
  entry:
	  movw	$0x1234,0x472			# warm boot
  ```
  对外声明了_start、entry符号，但是可以看到，这两个的值的区别，_start是RELOC后的entry标号值，我们看看entry标号值
  ```
  lumin@ubuntu:~/Documents/MIT-6.828/lab/obj/kern$ nm kernel| grep entry
  f010000c T entry
  f0112000 D entry_pgdir
  f0110000 D entry_pgtable
  ```
  entry值是0xf01000c，我们也可以找到RELOC宏的作用是减去一个KERNELBASE，即内核基地址，0xf0000000.关于这两个的差异放到最后再说吧，需要区分加载地址与链接地址（一开始我懵懵的）。

- Exercise:
  > What is on the stack?

  这个地方虽然描述很多，而且给了很多指引，但其实也比较简单：打印%esp后的内容，哪些值是属于栈的？含义是什么？（这里需要了解一下gcc关于栈的调用约定，可以参考lecture note2，介绍的非常简洁清晰）
  先不做实验，先想一想可能有什么？其实基本上没啥。因为我们直到进入bootmain之前才刚初始化了栈指针为0x7c00，跳入bootmain，栈指针-4用于存储返回地址，然后bootmain开始用栈，直到我们用一个函数指针跳转进入内核，这时候再-4，用于存储返回地址，那么是不是这样呢？我们看看
  跳转入bootmain之前：
  ```
  (gdb) b *0x7c45
  Breakpoint 1 at 0x7c45
  (gdb) c
  Continuing.
  The target architecture is assumed to be i386
  => 0x7c45:	call   0x7d15

  Breakpoint 1, 0x00007c45 in ?? ()
  ```
  这时候刚刚初始化完成esp，看一下寄存器
    ```
    (gdb) info reg
    eax            0x10	16
    ecx            0x0	0
    edx            0x80	128
    ebx            0x0	0
    esp            0x7c00	0x7c00
    ebp            0x0	0x0
    esi            0x0	0
    edi            0x0	0
    eip            0x7c45	0x7c45
    eflags         0x6	[ PF ]
    cs             0x8	8
    ss             0x10	16
    ds             0x10	16
    es             0x10	16
    fs             0x10	16
    gs             0x10	16
    ```
  esp没错，然后step一下，进入bootmain，再看一下esp：
    ```
    (gdb) si 1
    => 0x7d15:	push   %ebp
    0x00007d15 in ?? ()
    (gdb) info reg
    eax            0x10	16
    ecx            0x0	0
    edx            0x80	128
    ebx            0x0	0
    esp            0x7bfc	0x7bfc
    ebp            0x0	0x0
    esi            0x0	0
    edi            0x0	0
    eip            0x7d15	0x7d15
    eflags         0x6	[ PF ]
    cs             0x8	8
    ss             0x10	16
    ds             0x10	16
    es             0x10	16
    fs             0x10	16
    gs             0x10	16
    ```
  -4，看一下内容：
    ```
    (gdb) x/8x $esp
    0x7bfc:	0x00007c4a	0xc031fcfa	0xc08ed88e	0x64e4d08e
    0x7c0c:	0xfa7502a8	0x64e6d1b0	0x02a864e4	0xdfb0fa75
    ```
  值确系0x7c45+5（5字节的指令长）
  打断点直到跳入内核之前：
    ```
    (gdb) b *0x7d6b
    Breakpoint 2 at 0x7d6b
    (gdb) c
    Continuing.
    => 0x7d6b:	call   *0x10018

    Breakpoint 2, 0x00007d6b in ?? ()
    ```
  看一下栈：
    ```
    (gdb) x/10x $esp
    0x7bf0:	0x00000000	0x00000000	0x00000000	0x00007c4a
    0x7c00:	0xc031fcfa	0xc08ed88e	0x64e4d08e	0xfa7502a8
    0x7c10:	0x64e6d1b0	0x02a864e4
    ```
  从最初到现在用了16个字节，中间3个32位字都是0，最后一个就是刚才的返回地址
  step进入内核：
    ```
    (gdb) si 1
    => 0x10000c:	movw   $0x1234,0x472
    0x0010000c in ?? ()
    ```
  查看栈：
    ```
    (gdb) x/10x $esp
    0x7bec:	0x00007d71	0x00000000	0x00000000	0x00000000
    0x7bfc:	0x00007c4a	0xc031fcfa	0xc08ed88e	0x64e4d08e
    0x7c0c:	0xfa7502a8	0x64e6d1b0
    ```
  栈顶是返回地址，也就是跳转处的下一条指令地址0x7d6b+5，同时栈指针又-4
  至此栈内容解释就完了

# Postscript:

_关于entry标号和start标号的问题_
首先要明确的一点是，内核代码是放置在高地址空间部分的，这一部分可以参考《Understanding Linux Kernel》第二章关于地址空间的介绍。然而，我们并不一定有4GB的内存，或者无法支持这么高的地址，因此内存映射必不可少，可是我们到现在为止还没有配置内存映射怎么办呢？又变成了一个自举问题了。实验给出的方案是先手动给映射好了，把0xfxxxxxxx直接减去0xf0000000映射到0x0xxxxxxx。

但即便这一粗暴的做法，也要先配置好相应寄存器、页表啊，也就是说，我们总免不了要先在没有内存映射机制的状态下“裸奔”一小段儿。

我们来看看情况是怎样的。

首先看下objdump的kernel段情况：
```
lumin@ubuntu:~/Documents/MIT-6.828/lab/obj/kern$ objdump -h kernel

kernel:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001bf9  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       0000076b  f0101c00  00101c00  00002c00  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00003f85  f010236c  0010236c  0000336c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000019ca  f01062f1  001062f1  000072f1  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         00009300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .got          00000008  f0111300  00111300  00012300  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  6 .got.plt      0000000c  f0111308  00111308  00012308  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  7 .data.rel.local 00001000  f0112000  00112000  00013000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  8 .data.rel.ro.local 0000006c  f0113000  00113000  00014000  2**5
                  CONTENTS, ALLOC, LOAD, DATA
  9 .bss          00000648  f0113080  00113080  00014080  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 10 .comment      0000002b  00000000  00000000  000146c8  2**0
                  CONTENTS, READONLY
```

我们可以看到，它们的虚拟地址都是实际在内存上物理地址加上一个映射offset 0xf0000000。也就是说是这样：代码实际在内存上的低地址处，我们手动设置了标号_start的值，使之变为entry标号的实际物理地址
```
  .globl		_start
  _start = RELOC(entry)

  .globl entry
  entry:
	  movw	$0x1234,0x472			# warm boot
```
这样，我们的函数指针就可以跳转到合适的低地址处，也就是_start处，毕竟_start是默认入口点嘛。但是在开启分页前的这一小段要小心翼翼的写，要注意把标号全部都手动减去offset使之变为物理地址：
```
entry:
	movw	$0x1234,0x472			# warm boot

	# We haven't set up virtual memory yet, so we're running from
	# the physical address the boot loader loaded the kernel at: 1MB
	# (plus a few bytes).  However, the C code is linked to run at
	# KERNBASE+1MB.  Hence, we set up a trivial page directory that
	# translates virtual addresses [KERNBASE, KERNBASE+4MB) to
	# physical addresses [0, 4MB).  This 4MB region will be
	# sufficient until we set up our real page table in mem_init
	# in lab 2.

	# Load the physical address of entry_pgdir into cr3.  entry_pgdir
	# is defined in entrypgdir.c.
	movl	$(RELOC(entry_pgdir)), %eax
	movl	%eax, %cr3
	# Turn on paging.
	movl	%cr0, %eax
	orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
	movl	%eax, %cr0
```
就像这里的$(RELOC(entry_pgdir))，当打开PE位后，下面是jmp了一下
```
	# Now paging is enabled, but we're still running at a low EIP
	# (why is this okay?).  Jump up above KERNBASE before entering
	# C code.
	mov	$relocated, %eax
	jmp	*%eax
relocated:

	# Clear the frame pointer register (EBP)
	# so that once we get into debugging C code,
	# stack backtraces will be terminated properly.
	movl	$0x0,%ebp			# nuke frame pointer
```
有两个问题：
1. 首先就是这段代码注释写的，为啥我们现在仍然运行在low EIP上仍然ok呢？
2. 为啥这里要jmp一下？
第一个问题我们上面就解释过了，因为这段代码在内存上的实际位置就是在0x100000开始的低地址处，所以我们现在这样运行没毛病
第二个问题，为啥要jmp。注意，我们上面确实设置好了cr0，这意味着什么呢？
也就是说movl	%eax, %cr0 执行后我们就有内存影射了，访问0xfxxxxxxx就和访问0x0xxxxxxx一样了，但是注意，这时候我们的eip还是低地址处，因此我们要jmp一下，把processor刷新到现在的有内存映射访问状态，真正进入处理器的虚拟地址模式（虽然很弱），所以看起来有点蠢，手动跳转到下一条指令:)