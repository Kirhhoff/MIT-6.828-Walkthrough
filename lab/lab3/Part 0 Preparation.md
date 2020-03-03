# Preparation
这一部分我们首先根据xv6 book chapter 1的介绍分析一下xv6在这方面的源代码，捋请逻辑，然后的JOS代码分析以及补全就会相对轻松一点（JOS的实现更简化一些）

---
在main的尾声，我们已经完成了对内核内存空间映射的任务，后续需要做的就是手动配置第一个进程，将控制权交给它，进入进程空间。为了实现用户空间的种种特性，我们这一部分分析下xv6的源码，然后再进行实验。

在xv6中，这一过程分为两步
1. 设置好init进程的内容、状态
2. 交给调度器，开始进程的执行
```
int
main(void)
{
    ...
    userinit(); // first user process
    mpmain(); // finish this processor’s setup
}
```
分别对应到userinit和mpmain

userinit
---
```
void
userinit(void)
{
    struct proc *p;
    extern char _binary_initcode_start[], _binary_initcode_size[];

    p = allocproc();

    initproc = p;
    if((p−>pgdir = setupkvm()) == 0)
        panic("userinit: out of memory?");
    inituvm(p−>pgdir, _binary_initcode_start, (int)_binary_initcode_size);
    p−>sz = PGSIZE;
    memset(p−>tf, 0, sizeof(*p−>tf));
    p−>tf−>cs = (SEG_UCODE << 3) | DPL_USER;
    p−>tf−>ds = (SEG_UDATA << 3) | DPL_USER;
    p−>tf−>es = p−>tf−>ds;
    p−>tf−>ss = p−>tf−>ds;
    p−>tf−>eflags = FL_IF;
    p−>tf−>esp = PGSIZE;
    p−>tf−>eip = 0; // beginning of initcode.S

    safestrcpy(p−>name, "initcode", sizeof(p−>name));
    p−>cwd = namei("/");

    // this assignment to p−>state lets other cores
    // run this process. the acquire forces the above
    // writes to be visible, and the lock is also needed
    // because the assignment might not be atomic.
    acquire(&ptable.lock);
    p−>state = RUNNABLE;

    release(&ptable.lock);
}
```
- 首先，要有一个进程，于是首先分配一个进程结构体，allocproc，除了这里，fork时也会调用这个方法，内容如下：
    ```
    // Look in the process table for an UNUSED proc.
    // If found, change state to EMBRYO and initialize
    // state required to run in the kernel.
    // Otherwise return 0.
    static struct proc*
    allocproc(void)
    {
        struct proc *p;
        char *sp;

        acquire(&ptable.lock);

        for(p = ptable.proc; p < &ptable.proc[NPROC]; p++)
            if(p−>state == UNUSED)
                goto found;

        release(&ptable.lock);
        return 0;

        found:
            p−>state = EMBRYO;
            p−>pid = nextpid++;

            release(&ptable.lock);

            // Allocate kernel stack.
            if((p−>kstack = kalloc()) == 0){
                p−>state = UNUSED;
                return 0;
            }
            sp = p−>kstack + KSTACKSIZE;
            // Leave room for trap frame.
            sp −= sizeof *p−>tf;
            p−>tf = (struct trapframe*)sp;

            // Set up new context to start executing at forkret,
            // which returns to trapret.
            sp −= 4;
            *(uint*)sp = (uint)trapret;

            sp −= sizeof *p−>context;
            p−>context = (struct context*)sp;
            memset(p−>context, 0, sizeof *p−>context);
            p−>context−>eip = (uint)forkret;

            return p
        }
    ```
    这里做的是从全局进程表中找到一个未使用的位置，初始化它的状态后返回，具体这里需要初始化的状态都有：

    1. 进程状态
    2. 进程pid
    3. 每个进程各自的内核空间栈
    4. 在栈上预留Trapframe空间，内容留给调用者初始化，将tf指向这时的sp
    5. 预留出一个返回地址trapret（后面介绍）
    6. 预留出一个内核上下文context结构体（后面介绍），并设置它的sp和ip值，ip值固定为forkret，sp值为context起始处
    
    总的来说，除了对所有新进程都相同的内容（trapret，forkret），其他值都没有初始化，而交给调用者来初始化
- 有了进程，接下来就开始初始化进程的页表映射，setupkvm方法将初始化进程页表的内核页表部分，并将新的这个页目录返回：
    ```
    // Set up kernel part of a page table.
    pde_t*
    setupkvm(void)
    {
        pde_t *pgdir;
        struct kmap *k;

        if((pgdir = (pde_t*)kalloc()) == 0)
            return 0;
        memset(pgdir, 0, PGSIZE);
        if (P2V(PHYSTOP) > (void*)DEVSPACE)
            panic("PHYSTOP too high");
        for(k = kmap; k < &kmap[NELEM(kmap)]; k++)
            if(mappages(pgdir, k−>virt, k−>phys_end − k−>phys_start,
                    (uint)k−>phys_start, k−>perm) < 0) {
                freevm(pgdir);
                return 0;
            }
        return pgdir;
    }
    ```
    这一部分很明显就是在初始化一个页目录然后设置内核页映射接着返回了
- 设置好了内核页映射，接着设置用户空间页映射，这部分主要是把安装initcode到用户空间，而且这里的方法是只在这里用一次的：
    ```
    // Load the initcode into address 0 of pgdir.
    // sz must be less than a page.
    void
    inituvm(pde_t *pgdir, char *init, uint sz)
    {
        char *mem;

        if(sz >= PGSIZE)
            panic("inituvm: more than a page");
        mem = kalloc();
        memset(mem, 0, PGSIZE);
        mappages(pgdir, 0, PGSIZE, V2P(mem), PTE_W|PTE_U);
        memmove(mem, init, sz);
    }
    ```
    简单地设置了下页表然后就是拷贝，并且假设了initcode大小小于一个page
- 好了，进程有了，内核映射有了，源代码安装完了，最后只剩下进程栈内容的初始化和进程状态的初始化了，在代码上体现如下：
```
    p−>sz = PGSIZE;
    memset(p−>tf, 0, sizeof(*p−>tf));
    p−>tf−>cs = (SEG_UCODE << 3) | DPL_USER;
    p−>tf−>ds = (SEG_UDATA << 3) | DPL_USER;
    p−>tf−>es = p−>tf−>ds;
    p−>tf−>ss = p−>tf−>ds;
    p−>tf−>eflags = FL_IF;
    p−>tf−>esp = PGSIZE;
    p−>tf−>eip = 0; // beginning of initcode.S

    safestrcpy(p−>name, "initcode", sizeof(p−>name));
    p−>cwd = namei("/");

    // this assignment to p−>state lets other cores
    // run this process. the acquire forces the above
    // writes to be visible, and the lock is also needed
    // because the assignment might not be atomic.
    acquire(&ptable.lock);
    p−>state = RUNNABLE;

    release(&ptable.lock);
```
    总的来说，经过初始化以及原本就有的trapret和forkret后，栈上内容如下（后面根据调度器来解释栈上的内容用处）：
![](https://img-blog.csdnimg.cn/20200226190626153.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)

scheduler
---
```
// Common CPU setup code.
static void
mpmain(void)
{
    cprintf("cpu%d: starting %d\n", cpuid(), cpuid());
    idtinit(); // load idt register
    xchg(&(mycpu()−>started), 1); // tell startothers() we’re up
    scheduler(); // start running processes
}
```
总的来说mpmain的内容我们主要关注sheduler：
```
// Per−CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns. It loops, doing:
// − choose a process to run
// − swtch to start running that process
// − eventually that process transfers control
// via swtch back to the scheduler.
void
scheduler(void)
{
    struct proc *p;
    struct cpu *c = mycpu();
    c−>proc = 0;

    for(;;){
        // Enable interrupts on this processor.
        sti();

        // Loop over process table looking for process to run.
        acquire(&ptable.lock);
        for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
            if(p−>state != RUNNABLE)
                continue;

            // Switch to chosen process. It is the process’s job
            // to release ptable.lock and then reacquire it
            // before jumping back to us.
            c−>proc = p;
            switchuvm(p);
            p−>state = RUNNING;

            swtch(&(c−>scheduler), p−>context);
            switchkvm();

            // Process is done running for now.
            // It should have changed its p−>state before coming back.
            c−>proc = 0;
        }
        release(&ptable.lock);

    }
}
```
这里首先选择第一个找到的runnable进程，更新一下cpu的proc，接着切换到目标进程的页目录(switchuvm):
```
// Switch TSS and h/w page table to correspond to process p.
void
switchuvm(struct proc *p)
{
    if(p == 0)
        panic("switchuvm: no process");
    if(p−>kstack == 0)
        panic("switchuvm: no kstack");
    if(p−>pgdir == 0)
        panic("switchuvm: no pgdir");

    pushcli();
    mycpu()−>gdt[SEG_TSS] = SEG16(STS_T32A, &mycpu()−>ts,
    sizeof(mycpu()−>ts)−1, 0);
    mycpu()−>gdt[SEG_TSS].s = 0;
    mycpu()−>ts.ss0 = SEG_KDATA << 3;
    mycpu()−>ts.esp0 = (uint)p−>kstack + KSTACKSIZE;
    // setting IOPL=0 in eflags *and* iomb beyond the tss segment limit
    // forbids I/O instructions (e.g., inb and outb) from user space
    mycpu()−>ts.iomb = (ushort) 0xFFFF;
    ltr(SEG_TSS << 3);
    lcr3(V2P(p−>pgdir)); // switch to process’s address space
    popcli();
}
```
除去检查和TSS的设置，核心就是一句lcr3(V2P(p−>pgdir))，加载该进程的页目录物理地址到cr3，这对我们调度器的代码接着执行没有任何影响，因为我们现在就运行在内核态，所有进程的内核页映射都是一样的，切换一下对接下来的代码没影响，而且我们用的还是原来的内核栈(我们一定可以通过新进程的pgdir访问到原来进程的内核栈，因为这部分栈的页映射是包括在我们初始化阶段map的所有内核页的范围内的)，代码照常运行。

出来之后紧接着是这个：
```
p−>state = RUNNING;

swtch(&(c−>scheduler), p−>context);
```
重点是后面的swtch，这句话往后就不会返回了，正式切换到新进程的内核空间了，我们看一下源代码：
```
 .globl swtch
swtch:
    movl 4(%esp), %eax
    movl 8(%esp), %edx

    # Save old callee−save registers
    pushl %ebp
    pushl %ebx
    pushl %esi
    pushl %edi

    # Switch stacks
    movl %esp, (%eax)
    movl %edx, %esp

    # Load new callee−save registers
    popl %edi
    popl %esi
    popl %ebx
    popl %ebp
    ret
```
根据gcc的calling convention，4(%esp)为&(c−>scheduler)，8(%esp)为p->context，那么这里类似于：
```
eax=&(c−>scheduler)
edx=p->context
```
紧接着在栈上保存当前上下文，也就是调度器的上下文。这是谁的栈？
   
我们刚才分析过了，这里还是切换前的栈，中途也没有见到过设置栈指针的地方。

接着就到了切换栈了：
```
c->scheduler=esp
esp=p->context
```
这时候，就回到我们这张图了
![](https://img-blog.csdnimg.cn/20200226190626153.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
注意，这里图中的eip值为forkret，而且popl四句话后eip正好在栈顶，也就意味着，下一句ret后会跳转到forkret函数。

到这里我们停一下，我们做了什么呢？我们首先将scheduler的上下文信息保存在了切换前的栈上，接着把这时候的sp值存在了cpu->scheduler上，也就是这个cpu的scheduler成员中，等待下一次调度的时候恢复；同时，我们切换到了新进程的内核栈，取出了上面保存（在这里是初始化）的上下文信息，跳转到了forkret函数。

完美！

那么接下来我们看看forkret干了什么：
```
// A fork child’s very first scheduling by scheduler()
// will swtch here. "Return" to user space.
void
forkret(void)
{
    static int first = 1;
    // Still holding ptable.lock from scheduler.
    release(&ptable.lock);

    if (first) {
        // Some initialization functions must be run in the context
        // of a regular process (e.g., they call sleep), and thus cannot
        // be run from main().
        first = 0;
        iinit(ROOTDEV);
        initlog(ROOTDEV);
    }

    // Return to "caller", actually trapret (see allocproc).
}
```
对于第一次调用，会进程初始化等等，然后返回；对于不是第一次调用，直接就return。不管怎样，当要返回时栈顶都是trapret，因此下一步是跳转到trapret：
```
# Return falls through to trapret...
.globl trapret
trapret:
    popal
    popl %gs
    popl %fs
    popl %es
    popl %ds
    addl $0x8, %esp # trapno and errcode
    iret
```
注意此时，执行完一系列的语句时的栈状态如下：
![](https://img-blog.csdnimg.cn/20200226210323174.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
于是，我们一个iret（注意是iret不是ret，会把cs，ds，esp全部读出加载）就到了进程的入口点，开始用户态程序的执行

