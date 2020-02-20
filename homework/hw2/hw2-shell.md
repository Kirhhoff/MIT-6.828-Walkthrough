首先分析一下sh.c源码吧
主函数的逻辑大致如下
```
int
main(void)
{
  static char buf[100];
  int fd, r;

  // Read and run input commands.
  while(getcmd(buf, sizeof(buf)) >= 0){
    if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
      // Clumsy but will have to do for now.
      // Chdir has no effect on the parent if run in the child.
      buf[strlen(buf)-1] = 0;  // chop \n
      if(chdir(buf+3) < 0)
        fprintf(stderr, "cannot cd %s\n", buf+3);
      continue;
    }
    if(fork1() == 0)
      runcmd(parsecmd(buf));
    wait(&r);
  }
  exit(0);
}
```
通过getcmd获得用户输入的字符串放入buf中，通过parsecmd方法解析得到cmd对象，fork出一个子进程来执行，而解析字符串得到cmd对象（也就是parsecmd方法）是lab已经预先实现了的，我们需要做的是实现如先执行cmd，也就是runcmd方法。
关于parsecmd的分析放在末尾，最为可选的吧，我们来看看要解决hw的问题要怎么实现runcmd
# runcmd
首先我们要先了解下这个cmd的结构
```
// All commands have at least a type. Have looked at the type, the code
// typically casts the *cmd to some specific cmd type.
struct cmd {
  int type;          //  ' ' (exec), | (pipe), '<' or '>' for redirection
};

struct execcmd {
  int type;              // ' '
  char *argv[MAXARGS];   // arguments to the command to be exec-ed
};

struct redircmd {
  int type;          // < or > 
  struct cmd *cmd;   // the command to be run (e.g., an execcmd)
  char *file;        // the input/output file
  int flags;         // flags for open() indicating read or write
  int fd;            // the file descriptor number to use for the file
};

struct pipecmd {
  int type;          // |
  struct cmd *left;  // left side of pipe
  struct cmd *right; // right side of pipe
};
```
可以看出，cmd模拟了“基类”，主要是这三个后继：可执行命令，重定向命令，管道命令。我们从最基本的execcmd开始
```
  switch(cmd->type){
  default:
    fprintf(stderr, "unknown runcmd\n");
    _exit(-1);

  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      _exit(0);
    fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    break;
```
这里为我们空出了位置，我们要做的就是根据这个拿到的ecmd指针来执行这个命令
ecmd结构简单，就只有type和argv两个字段，argv和main函数里那个参数的含义相同，index=0处为程序名，后面全是一个个的参数，我们要做的，就是调用execv来执行它。关于execv的说明我们可以man 3 exec查看：
> int execv(const char *path, char *const argv[]);
> The execv(), execvp(), and execvpe()  functions  provide  an  array  of pointers  to  null-terminated  strings that represent the argument list available to the new  program.   The  first  argument,  by  convention, should  point  to the filename associated with the file being executed. The array of pointers must be terminated by a null pointer.

那么这一段代码如下：
```
// Your code here ...
if(execv(ecmd->argv[0],ecmd->argv)<0)
    fprintf(stderr,"Fail to execute \"%s\" with errno: %d\n",ecmd->argv[0],errno);  
```
对execv进行简单的错误检验。另外这里要说的是，一旦调用了execv，调用的程序就会覆盖当前的进程，不发生错误就不会返回了，这也是为什么一开始我们要fork一个子进程来runcmd，因为原来的进程要留着给用户接着输入命令，不能覆盖
> RETURN VALUE
       The exec() functions return only if an error has occurred.  The  return
       value is -1, and errno is set to indicate the error.

接下来是重定向命令
说实话这个重定向命令花了我两三个小时，因为一开始不知道怎么把execv调用的程序的stdin，stdout重定向到文件（甚至一开始还想着自己开缓存区copy...，但是要处理变长、溢出等等想想不太可能是这样就放弃了）后来终于找到了方法，dup2
首先要明确的一点是，由于execv是将用调用的程序来覆写当前的进程，许多进程的状态信息都与覆写前的相同，而输入输出也在其中，所以我们要在execv之前将进程的输出输入（输出）由标准输入（标准输入）重定向到我们要的文件:
```
struct redircmd {
  int type;          // < or > 
  struct cmd *cmd;   // the command to be run (e.g., an execcmd)
  char *file;        // the input/output file
  int flags;         // flags for open() indicating read or write
  int fd;            // the file descriptor number to use for the file
};
```
redircmd中制定了文件名、文件的读写权限、要替换的是stdin还是stdout
OK，我们来看下dup2，它的定义如下：
```
int dup2(int fd,int fd2);
```
我们可以这样使用：
```
dup2(file_fd,fileno(stdout));
```
这样就可以将stdout实际的指向由终端转向file_fd所指的文件，借助这个函数，我们得以实现重定向命令：
```
// Your code here ...
int file2rw_fd=open(rcmd->file,rcmd->flags,S_IRUSR|S_IWUSR);

if(file2rw_fd<0)//check file descriptor
    fprintf(stderr,"Fail to open file %s\n",rcmd->file);

if(dup2(file2rw_fd,rcmd->fd)<0){//redirect 
    fprintf(stderr,"Fail to redirect stdin/stdout to file %s\n",rcmd->file);            
    _exit(-1);
    
runcmd(rcmd->cmd);
```
最后是管道命令
```
struct pipecmd {
  int type;          // |
  struct cmd *left;  // left side of pipe
  struct cmd *right; // right side of pipe
};
```
按照重定向命令的经验，我们应该执行命令时fork，父进程执行left cmd，子进程执行right cmd，同时使left cmd的标准输出与right cmd的标准输入连接起来，但是这个单凭dup2似乎搞不定，因为之前我们事先知道了文件描述符，而这里在父进程中我们如何知道子进程的标准输入fd，在子进程中我们如何知道父进程的标准输出fd？根=根据提示，我们找到了man 2 pipe:
```
int pipe(int pipefd[2]);
```
> DESCRIPTION
pipe()  creates  a pipe, a unidirectional data channel that can be used for interprocess communication.  The array pipefd is used to return two file  descriptors  referring to the ends of the pipe. pipefd[0] refers to the read end of the pipe.  pipefd[1] refers to the write end of the pipe. Data  written  to  the write end of the pipe is buffered by the kernel until it is read from the read end of  the pipe.

根据pipe的描述，我们可以如下实现：
```
pcmd = (struct pipecmd*)cmd;

const int read_port=0,write_port=1;
int pipefd[2];//

if(pipe(pipefd)<0){
    fprintf(stderr,"Fail to create pipe between P/C processes.\n");
}

if(fork1()==0){//child process
    
    close(pipefd[write_port]);

    if(dup2(pipefd[read_port],fileno(stdin))<0)
    fprintf(stderr,"Fail to redirect stdin from parent process.\n");

    runcmd(pcmd->right);

}else{//parent process

    close(pipefd[read_port]);

    if(dup2(pipefd[write_port],fileno(stdout))<0)
    fprintf(stderr,"Fail to redirect stdout to child process.\n");

    runcmd(pcmd->left);

}
```
在父/子进程中分别关闭掉不需要的端口号，而后分别进行各自的重定向，进程间的同步由内核负责，不用咱们操心了。
