# Part One: System call tracing
这部分相当于在原先的系统调用附近加个hook，首先为了方便索引名字，我们定义一个名字数组：
```
char* sysnames[]={
  "",
  "fork",
  "exit",
  "wait",
  "pipe",
  "read",
  "kill",
  "exec",
  "fstat",
  "chdir",
  "dup",
  "getpid",
  "sbrk",
  "sleep",
  "uptime",
  "open",
  "write",
  "mknod",
  "unlink",
  "link",
  "mkdir",
  "close",
  "date",
  "dup2",
};
```
注意数组的第一个元素空出来了，因为系统调用号是从1开始数的。由于系统调用号是存在%eax中的，hook函数就很自然了：
```
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    curproc->tf->eax = syscalls[num]();
    cprintf("%s -> %d\n",sysnames[num],curproc->tf->eax);
  } else {
    cprintf("%d %s: unknown sys call %d\n",
            curproc->pid, curproc->name, num);
    curproc->tf->eax = -1;
  }
```
也就是插一句cprintf，效果如下：
```
exec -> 0
open -> 0
dup -> 1
dup -> 2
iwrite -> 1
nwrite -> 1
iwrite -> 1
twrite -> 1
:write -> 1
 write -> 1
swrite -> 1
twrite -> 1
awrite -> 1
rwrite -> 1
twrite -> 1
iwrite -> 1
nwrite -> 1
gwrite -> 1
 write -> 1
swrite -> 1
hwrite -> 1

write -> 1
fork -> 2
exec -> 0
open -> 3
close -> 0
$write -> 1
 write -> 1
```
# Part One Challenge
打印参数需要清楚每个函数的参数情况，因此相对麻烦一点，这里只对用到的系统调用进行了参数打印：
```
void print_syscall_args(int sysno){
  char* path=0;
  uint argv,char_addr=0;
  int mode,fd,n;

  switch (sysno)
  {
  case SYS_exec:
    argint(1,(int*)&argv);
    fetchint(argv,(int*)&char_addr);
    cprintf(" path=\"%s\"",(char*)char_addr);
    for(int i=1;;i++){
      fetchint(argv+4*i,(int*)&char_addr);
      if(char_addr==0)
        goto done;
      else
        cprintf(" \"%s\"",(char*)char_addr);
    }
  case SYS_open:
    argstr(0,&path);
    argint(1,&mode);
    cprintf(" path=\"%s\" mode=%d",path,mode);
    break;
  case SYS_write:
    argint(0,&fd);
    argint(1,(int*)&char_addr);
    argint(2,&n);
    cprintf(" fd=%d str=0x%x bytes=%d",fd,char_addr,n);
    break;
  case SYS_close:
  case SYS_dup:
    argint(0,&fd);
    cprintf(" fd=%d",fd);
    break;
  case SYS_dup2:
    argint(0,&fd);
    cprintf(" old_fd=%d",fd);
    argint(1,&fd);
    cprintf(" new_fd=%d",fd);
    break;
  default:
    break;
  }
  done:
    cprintf("\n");
}
```
hook函数部分变为：
```
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    curproc->tf->eax = syscalls[num]();
    cprintf("%s -> %d args:",sysnames[num],curproc->tf->eax);
    print_syscall_args(num);
  } else {
    cprintf("%d %s: unknown sys call %d\n",
            curproc->pid, curproc->name, num);
    curproc->tf->eax = -1;
  }
```
效果如下：
```
exec -> 0 args: path="/init"
open -> 0 args: path="console" mode=2
dup -> 1 args: fd=0
dup -> 2 args: fd=0
iwrite -> 1 args: fd=1 str=0x2f9a bytes=1
nwrite -> 1 args: fd=1 str=0x2f9a bytes=1
iwrite -> 1 args: fd=1 str=0x2f9a bytes=1
twrite -> 1 args: fd=1 str=0x2f9a bytes=1
:write -> 1 args: fd=1 str=0x2f9a bytes=1
 write -> 1 args: fd=1 str=0x2f9a bytes=1
swrite -> 1 args: fd=1 str=0x2f9a bytes=1
twrite -> 1 args: fd=1 str=0x2f9a bytes=1
awrite -> 1 args: fd=1 str=0x2f9a bytes=1
rwrite -> 1 args: fd=1 str=0x2f9a bytes=1
twrite -> 1 args: fd=1 str=0x2f9a bytes=1
iwrite -> 1 args: fd=1 str=0x2f9a bytes=1
nwrite -> 1 args: fd=1 str=0x2f9a bytes=1
gwrite -> 1 args: fd=1 str=0x2f9a bytes=1
 write -> 1 args: fd=1 str=0x2f9a bytes=1
swrite -> 1 args: fd=1 str=0x2f9a bytes=1
hwrite -> 1 args: fd=1 str=0x2f9a bytes=1

write -> 1 args: fd=1 str=0x2f9a bytes=1
fork -> 2 args:
exec -> 0 args: path="sh"
open -> 3 args: path="console" mode=2
close -> 0 args: fd=3
$write -> 1 args: fd=2 str=0x3f7a bytes=1
 write -> 1 args: fd=2 str=0x3f7a bytes=1
```

# Part Two: Date system call
这一部分是要求使用cmos固件提供的接口实现date系统调用，比较简单：
```
int sys_date(void){
  struct rtcdate r;
  struct rtcdate* arg;

  // argument retrieve and check
  if(argptr(0,(char**)&arg,sizeof(r))<0)
    return -1;
  
  // get time
  cmostime(arg);

  return 0;
}
```
要修改的文件比较多

首先要对用户可见，因此在user.h中添加用户的系统调用接口：
```
int date(struct rtcdate*);
```
但这个函数并不是定义在一个C文件中，而是在usys.S中通过汇编指令定义的，因此需要在usys.S中增加实现：
```
#define SYSCALL(name) \
  .globl name; \
  name: \
    movl $SYS_ ## name, %eax; \
    int $T_SYSCALL; \
    ret
SYSCALL(date)
```
再接着就是在内核文件中添加SYS_date宏：
```
#define SYS_date   22
```
以及extern声明函数（sys_date定义在别处）：
```
extern int sys_date(void);
```
最后函数指针数组中添加对应项：
```
[SYS_date]   sys_date,
```

添加date.c到Makefile中：
```
UPROGS=\
	_cat\
	_echo\
	_forktest\
	_grep\
	_init\
	_kill\
	_ln\
	_ls\
	_mkdir\
	_rm\
	_sh\
	_stressfs\
	_usertests\
	_wc\
	_zombie\
	_date\
```
最后是date.c定义测试代码：
```
#include "types.h"
#include "user.h"
#include "date.h"

int
main(int argc, char *argv[])
{
  struct rtcdate r;

  if (date(&r)){
    printf(2, "date failed\n");
    exit();
  }

  // your code to print the time in any format you like...
  printf(1,"UTCtime: %d-%d-%d %d:%d:%d\n",r.year,r.month,r.day,r.hour,r.minute,r.second);
  
  exit();
}
```
测试：
```
$ date
UTCtime: 2020-3-3 10:23:40
```

# Part Two Challenge
这部分要求是效仿之前date的模式，添加dup2系统调用，这个函数在实现shell的时候用过一次，这里亲手实现一下：
```
int
sys_dup2(void)
{
  struct file *oldf,*newf;
  int old_fd,new_fd;
  struct proc* curproc;

  // obtain target file pointer
  if(argfd(0, 0, &oldf) < 0)
    return -1;

  // retrieve arguments
  if(argint(0,&old_fd)<0||argint(1,&new_fd)<0)
    return -1;
  
  // dup only if new_fd doesn't equal old_fd
  if(old_fd!=new_fd){
    curproc=myproc();
    newf=curproc->ofile[new_fd];
    // dup only if newf doesn't equal oldf
    if(newf!=oldf){
      // if newf is non-null, close it
      if(newf!=0)
        fileclose(newf);
      
      // update necessary records
      curproc->ofile[new_fd]=oldf;
      filedup(oldf);    
    }
  }

  return new_fd;
}
```
大体上来说，就是使得new_fd也指向old_fd指向的文件，并执行必要的关闭操作，具体要实现的功能可以参照man 2 dup2。测试就随便创建了个文件：
```
#include "fcntl.h"
#include "types.h"
#include "user.h"

int
main(int argc, char *argv[]){
    
    if(open("console", O_RDWR) < 0){
        mknod("console", 1, 1);
        open("console", O_RDWR);
    }
    
    dup2(0,1);  // stdout
    dup2(0,2);  // stderr

    printf(1,"dup2 test passed!\n");
    exit();
}
```
注意这里的stdout和stderr在dup2之前就已经是指向控制台了，可以在init中看到，测试效果如下：
```
$ dup2
dup2 test passed!
```