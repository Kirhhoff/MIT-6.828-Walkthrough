# Part One: Eliminate allocation from sbrk()

回顾exec调用，我们并没有调用过growproc，而是直接allocuvm进行的内存分配，因为我们在初始化阶段可以认为代码段和全局数据段是必然要用到的，因此不需要lazy allocation.

内存分配调用过程：
```
malloc:
    sbrk:
        sys_sbrk:
            growproc:
                allocuvm:
                    ...    
```
而我们要做的是在growproc中取消页面分配，因此初始化阶段直接调用allocuvm不会造成触发lazy allocation.

仔细定位一下可以发现，malloc的调用发生在sh.c的runcmd过程中用于动态创建execcmd对象：
```
struct cmd*
execcmd(void)
{
  struct execcmd *cmd;

  cmd = malloc(sizeof(*cmd));
  memset(cmd, 0, sizeof(*cmd));
  cmd->type = EXEC;
  return (struct cmd*)cmd;
}
```
也就是说这个page fault是进程在执行echo前触发的。

# Part Two: Lazy allocation

要解决lazy allocation，我们需要在page fault处加个“hook”检查是否是起源于lazy allocation，如果是则分配返回重新执行指令.

检查原因：
```
// check whether it raises from lazy page allocation
static inline 
int check_lazy_page(uint proc_sz,struct trapframe* tf,uint addr){
  return tf->trapno==T_PGFLT // check trapno
          &&addr<PGROUNDUP(proc_sz) // check whether addr belongs to the process
          &&!(~tf->err&(PTE_U|PTE_W)); // check permission(in case of frobidden area like guarded stack page)
}
```
进行page分配：
```
int lazy_allocation(pde_t* pgdir,uint va){
  void* pa;

  // allocation physical page
  if((pa=kalloc())==0)
    return -1;
  
  // clear memory
  memset(pa,0,PGSIZE);

  // mapping
  if(mappages(pgdir,(void*)PGROUNDDOWN(va),PGSIZE,V2P(pa),PTE_U|PTE_W)<0)
    return -1;
  
  return 0;
}
```
# Challenge
> Handle negative sbrk() arguments. Handle error cases such as sbrk() arguments that are too large. Verify that fork() and exit() work even if some sbrk()'d address have no memory allocated for them. Correctly handle faults on the invalid page below the stack. Make sure that kernel use of not-yet-allocated user addresses works -- for example, if a program passes an sbrk()-allocated address to read().

我们在growproc中粗暴地注释掉了allocuvm，而实际上相对于我们在lazy alloc中的分配工作，我们还缺少了必要的边界检查，因此我们补上：
```
  uint sz,newsz;
  struct proc *curproc = myproc();

  sz = curproc->sz;
  if(n > 0){
    newsz=sz+n;
    if(newsz>=KERNBASE||newsz<sz)
      return-1;
    sz+=n;  
  } else if(n < 0){
    if((sz = deallocuvm(curproc->pgdir, sz, sz + n)) == 0)
      return -1;
  }
```
我们分析下各种情况：
1. negative sbrk() arguments：这个不在我们的condition case中
2. sbrk() arguments that are too large：这个在通过我们新加的边界检查可以避免
3. Verify that fork() and exit() work even if some sbrk()'d address have no memory allocated for them. 这个我们必需要对fork进行修改，在fork中copyuvm时存在对页present的检查：
    ```
    if(!(*pte & PTE_P))
      panic("copyuvm: page not present");
    ```
    我们改动为：
    ```
    if(!(*pte & PTE_P))
      continue;
    ```
    在echo中添加测试代码：
    ```
    // allocate but not access
    char* p=sbrk(10);

    if(fork()==0){
        // write it in child process
        p[0]='a';
    }else
        // parent process wait for child process exit
        wait();
    ```
    可以通过测试
4. Make sure that kernel use of not-yet-allocated user addresses works -- for example, if a program passes an sbrk()-allocated address to read(). 我们稍微修改一下echo.c，加上测试代码：
    ```
    int fd=open("echo.c",O_RDONLY);
    read(fd,sbrk(11),10);    
    ```
    可以正常通过，因为内核对页的访问也是通过page fault路径，没有区别