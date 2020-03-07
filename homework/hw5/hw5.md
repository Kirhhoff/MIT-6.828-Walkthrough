# Homework 5: xv6 CPU alarm

这部分的工作是提供一个系统调用使得用户能够为进程定义一个periodic callback函数，使得每过一段时间，这个callback能够自动执行。因此我们要先介绍一下时钟中断，再据此设计我们的新系统调用

时钟中断
---
每当经过一个tick时钟中断会触发一次，更新相应的信息，而具体一个tick是多久因计算机而异，如对于普通计算机，要求一秒钟起码要有60个tick来维持60hz的刷新率。每次触发中断都要进行上下文保存、切换以及tlb刷新，因此一个tick太短会影响总体的吞吐量，而一个tick太长会降低系统与用户的交互性，我们可以简单地算个小例子：加入CPU时钟频率为2GHz，每秒钟200个tick，平均每次处理时钟中断的overhead为2000个时钟周期，那么平均而言，时钟中断带来的overhead与CPU的总吞吐量比例为：200*2000/2G=1/5000，大体上也还能接受。


alarm
---
因此我们需要做的就是在每个时钟中断到来之时，检查当前进程（因为现在我们只有这一个进程）的handler是否为空，如果不为空，检查一下是否经过了足够长的interval(alarmtick)，满足的话进行必要的检查以及状态更新，接着修改原先的trapframe和栈来使得内核态回到用户态时不再是回到原来触发时钟中断的地方，而是跳转到handler函数，而handler函数执行完后自动return（通过合理的栈内容安排）到当前进程被中断的地方。

首先我们按照之前的方法在必需的各处添加alarm系统调用的声明，对于用户的声明如下：
```
int alarm(int ntick,void handler());
```
其他在内核态的声明我们就不列出了，我们直接看alarm系统调用的实现:
```
int sys_alarm(void)
{
  int ntick;
  void (*handler)();

  // retrieve parameters
  // 0-ntick 1-handler
  if(argint(0,&ntick)<0||argptr(1,(char**)&handler,sizeof(handler))<0)
    return -1;
  
  // check parameters
  if(ntick<=0||!handler)
    return -1;

  // update myproc
  myproc()->alarmticks=ntick; 
  myproc()->alarmhandler=handler;
  myproc()->lastalarmtick=ticks;

  return 0;
}
```
其中需要在proc结构体中增加的member:
```
  int alarmticks;              // tick interval to invoke alarmhandler 
  void (*alarmhandler)();      // alarm handler to invoke per tick interval
  int lastalarmtick;           // tick at which last invocation to alarmhandler
```
OK，用来注册的系统调用实现好了，剩下的就是在时钟中断处增加检查代码了：
```
    // check there is a current process and
    // it comes from userspace
    if(myproc()!=0&&(tf->cs&0x3)==0x3){
      // check alarm expired
      if(ticks>=myproc()->lastalarmtick+myproc()->alarmticks){
        // update info 
        myproc()->lastalarmtick=ticks;

        // simulate pushing stopped address onto
        // userspace stack such that, when alarm
        // handler returns it jumps back to where
        // the user process was stopped
        tf->esp-=4;
        *((uint*)tf->esp)=tf->eip;
        
        // modify iret address to 
        // return to alarm handler rather than 
        // where the process is stopped
        tf->eip=(uint)myproc()->alarmhandler;
      }
```
在这里，我们把eip修改成了handler地址，但是要怎么回到用户地址呢？很简单，在用户栈上再push上去原先的eip即可。以此便实现了alarm中断

# Challenge
> 1. Save and restore the caller-saved user registers around the call to handler.
> 2. Prevent re-entrant calls to the handler -- if a handler hasn't returned yet, don't call it again.
> 3. Assuming your code doesn't check that tf->esp is valid, implement a security attack on the kernel that exploits your alarm handler calling code.

这部分challenge困扰了我两天，找了很多参考也无果，后来还是看了一位救星在github上的[解答](https://github.com/ldaochen/OS#xv6-cpu-alarm)才醒悟，佩服这人的脑洞，惭愧惭愧

在之前我们的实现有三个很大的问题：
1. 我们无法确保caller-saved registers不被破坏。在跳转到handler时，handler默认caller是已经保存且会自行恢复需要保护的caller-saved registers的，而这里我们并没有做这一步，因此很有可能这部分寄存器会被handler破坏，相当于我们在进程中断处不知道的情况下把它的寄存器破坏了，因此我们要在进入handler之前保存这些寄存器，在handler返回前恢复这些寄存器
2. 我们没有确保handler是不可重入的。什么意思呢？在handler尚未执行完时，我们没有确保进程不会再次进入handler。就我们这个例子来说，假如在进入handler，刚刚打印完"ala"时进程被挂起，很久都没有恢复，而此时又过了一个alarmtick周期，内核会使当前进程再次进入handler，也就是所谓的“重入”，打印“alarm!”，如此一来，控制台的输出就变成了“alaalarm!rm!”，这不是我们想看到的，因此我们要确保handler不可重入
3. 我们没有确保tf->esp的合法性。在时钟中断处，我们通过修改用户栈来安排执行流程，修改用户栈本身是通过tf->esp进行的，假如tf->esp不合法（如恶意用户设为内核某地址），内核可能会被破坏，因此我们需要防范这一点

第3条很容易实现，我们只要在时钟中断处增加一个检查即可，但1，2两条都面临一个问题：如何向用户态注入代码？

对1来说，保存寄存器简单，我们可以在内核态时钟中断处执行，但恢复呢？我们的执行流是handler->return to->user process，如果想要执行这一操作，需要把它变成：handler->return to->restore code->return to/jump to->user process。问题来了，这部分代码是放在内核态还是用户态呢？

首先，这段代码无法放在内核态，因为handler是用户自定义的函数，结尾必然只是一条return，而return只能向同权限级或更低权限级代码跳转（我曾诉诸于call gate等等方法实现，使得这条return能够跳转到内核态，但return必须要有参数，除非篡改用户代码否则用户代码只会是一条普通的return，而篡改用户代码又不现实，而且有安全隐患，比如用户蓄意设置栈然后调用来跳转到内核态等等）。

而后我想，能否直接在时钟中断处由内核直接调用handler？想想也不行，如此一来，运行级别就成了内核态，很容易引起恶意用户的蓄谋攻击。

总而言之，寄存器的恢复就无法在内核态进行了，那么能否放在用户态库中呢？也不太行。因为放在用户态，那么内核的编译要么就得依赖用户态库代码，这显然是荒唐的；但又不能把用户态库的这部分恢复寄存器代码地址写死在内核中，因为代码地址可能会重定向等等。因此，在用户态库中实现也不行。

想到这里，也查了许多资料，都没能得以解决，十分头痛。

最后是在开头那个小哥那里找到了解决方案，在用户栈上hard code一段代码，使得handler返回跳转到那里，然后在那里触发一个新的系统调用，在那个系统调用中恢复寄存器等等要进行的操作，最后再从栈上这段代码跳回到用户进程的中断处。

五体投地哈哈哈~

那么我们来安排一下这段栈。首先要知道这段“栈上代码”的长度以及其二进制代码，我们写一段编译得到：
```
.globl before_userhandler_return
before_userhandler_return:
  movl $25,%eax
801059ba:	b8 19 00 00 00       	mov    $0x19,%eax
  add $0xc,%esp
801059bf:	83 c4 0c             	add    $0xc,%esp
  int $64
801059c2:	cd 40                	int    $0x40
801059c4:	c3                   	ret    
801059c5:	66 90                	xchg   %ax,%ax
```
这里0x19(25)是新系统调用的号码，int $0x40是触发系统调用中断所以我们这段栈上代码只是负责触发一次系统调用，在那个系统调用中完成寄存器的恢复等等，等中断结束，返回到我们刻意安排的地址，也就是最开始用户进程被打断的地方。

为什么我们不直接在这段栈上代码中恢复寄存器而非要触发一次系统调用在那里处理呢？主要是因为这部分是hard code的，不宜过长，容易出问题，也难理解，而且在系统调用中处理，我们可以在系统调用中添加其他功能，比如解决2，更新可重入信息，假如要在栈上“硬”写这些代码的二进制，简直不可想象。。因此这里也是为了扩展性着想。

那么我们来安排栈：
```
// original stopped point
*(uint*)(tf->esp-4)=tf->eip;
// save caller-saved registers
*(uint*)(tf->esp-8)=tf->eax;
*(uint*)(tf->esp-12)=tf->edx;
*(uint*)(tf->esp-16)=tf->ecx;
// disassembly for:
// mov    $0x19,%eax
// add    $0xc,%esp
// int    $0x40
// ret
*(uint*)(tf->esp-20)=0x66c340cd;
*(uint*)(tf->esp-24)=0x0cc48300;
*(uint*)(tf->esp-28)=0x000019b8;
// start address of thes trick codes
*(uint*)(tf->esp-32)=tf->esp-28;
// update esp after such a bunch of pushes
tf->esp=tf->esp-32;        
// iret address
tf->eip=(uint)myproc()->alarmhandler;
```
经过如此安排后栈内容如下（从下到上地址升高）：
```
|user content|<-original esp
|stopped point of user space|
|saved eax|
|saved edx|
|saved ecx|
|hard code: ret|
|hard code: int $40|
|hard code: add $0xc,%esp|
|hard code: mov $0x19,%eax|<-hard code start
|hard code start|<-current esp
```
注意，hard code其实只占3个4字节字，但是有四行代码，这里为了美观所以看起来像是4个4字节字。

执行过程以及栈内容如下：
1. handler执行完ret指令时:
    ```
    |user content|<-original esp
    |stopped point of user space|
    |saved eax|
    |saved edx|
    |saved ecx|
    |hard code: ret|
    |hard code: int $40|
    |hard code: add $0xc,%esp|
    |hard code: mov $0x19,%eax|<-current eip & current esp
    ```
2. 执行完add $0xc,%esp指令后，执行int $40指令前：
    ```
    |user content|<-original esp
    |stopped point of user space|
    |saved eax|
    |saved edx|
    |saved ecx|<-current esp
    |hard code: ret|
    |hard code: int $40|<-current eip
    ```
3. int结束后，ret前：
    ```
    |user content|<-original esp
    |stopped point of user space|<-current esp
    |saved eax|
    |saved edx|
    |saved ecx|
    |hard code: ret|<-current eip   
    ```
4. ret结束：
    ```
    |user content|<-current esp
    ```

综上，栈的安排就结束了，但是，我们还有一部分工作是在新的系统调用中进行的：
```
int sys_alarmhandler_wrapper(void){
  int ret,eax,edx,ecx;
  // retrieve saved register contents
  argint(0,&ecx);
  argint(1,&edx);
  argint(2,&eax);
  // move esp
  myproc()->tf->esp+=12;
  
  // restore saved registers
  asm volatile("movl %0,%%eax"::"S"(eax));
  asm volatile("movl %0,%%edx"::"S"(edx));
  asm volatile("movl %0,%%ecx"::"S"(ecx));
  asm volatile("movl %0,%%eax":"=a"(ret));

  // to maintain eax
  return ret;
}
```
其中的tf->esp+=12就是为了使esp指向正确的返回地址。这里有一些地方需要注意：
1. 要先提取全部参数再恢复，因为eax是caller-saved register，执行argint时会破坏eax，所以不能先恢复eax
2. asm volatile尽量指定除eax、edx和ecx外的寄存器来当中间寄存器（这里选了esi），避免被破坏出bug
3. 这里的ret值是为了保持函数声明格式以放入系统调用函数指针数组，又为了维持eax不变还能通过编译，所以把eax值取出进行返回

最后，为了实现不可重入，我们在proc中加一个字段：
```
  int inalarmhandler;          // info about reentrant
```
trap中进入handler前：
```
      if((ticks>=myproc()->lastalarmtick+myproc()->alarmticks)
          &&!myproc()->inalarmhandler){
        ...
        myproc()->inalarmhandler=1;
        ...
      }
```
在我们的新系统调用中：
```
myproc()->inalarmhandler=0;
```
即可