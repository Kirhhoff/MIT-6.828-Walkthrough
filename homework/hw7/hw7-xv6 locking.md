# Homework 7: xv6 locking
这一部分的问题因为过于简单就直接brain experiment了

Don't do this
---
> Make sure you understand what would happen if the xv6 kernel executed the following code snippet:
>
>  ```
> struct spinlock lk;
>  initlock(&lk, "test lock");
>  acquire(&lk);
>  acquire(&lk);
这很明显，应当是报already hold错误，拿着锁还获取锁，说的就是你！>>

Interrupts in ide.c
---
> Let's see what happens if we turn on interrupts while holding the ide lock. In iderw in ide.c, add a call to sti() after the acquire(), and a call to cli() just before the release(). Rebuild the kernel and boot it in QEMU. Chances are the kernel will panic soon after boot; try booting QEMU a few times if it doesn't.
> 
> **Submit: Explain in a few sentences why the kernel panicked. You may find it useful to look up the stack trace (the sequence of %eip values printed by panic) in the kernel.asm listing.**

这个很简单，在拿着锁的时候如果开了中断，那么就会有两种情况:
1. 中断处理程序有可能需要获取这个锁
2. 中断处理程序不会获取这个锁

前者就可能引发死锁，后者则不会引发死锁。而我们可以看到，对磁盘驱动程序：
```
// Interrupt handler.
void
ideintr(void)
{
  struct buf *b;

  // First queued buffer is the active request.
  acquire(&idelock);

  if((b = idequeue) == 0){
    release(&idelock);
    return;
  }
  idequeue = b->qnext;

  // Read data if needed.
  if(!(b->flags & B_DIRTY) && idewait(1) >= 0)
    insl(0x1f0, b->data, BSIZE/4);

  // Wake process waiting for this buf.
  b->flags |= B_VALID;
  b->flags &= ~B_DIRTY;
  wakeup(b);

  // Start disk on next buf in queue.
  if(idequeue != 0)
    idestart(idequeue);

  release(&idelock);
}
```
在中断handler中再次获取了idelock。因此，如果我们获取了idelock后打开中断，就有极有可能deadlock在中断handler中。

Interrupts in file.c
---
> Now let's see what happens if we turn on interrupts while holding the file_table_lock. This lock protects the table of file descriptors, which the kernel modifies when an application opens or closes a file. In filealloc() in file.c, add a call to sti() after the call to acquire(), and a cli() just before each of the release()es.
> 
> **Submit: Explain in a few sentences why the kernel didn't panic. Why do file_table_lock and ide_lock have different behavior in this respect?**
>
> You do not need to understand anything about the details of the IDE hardware to answer this question, but you may find it helpful to look at which functions acquire each lock, and then at when those functions get called.

这里我们可以看到，file的锁恰好是对应于刚才分析的第2种情况，不会造成死锁。我们看一下都在哪里获取了锁：
```
// Allocate a file structure.
struct file*
filealloc(void)
{
  struct file *f;

  acquire(&ftable.lock);
  for(f = ftable.file; f < ftable.file + NFILE; f++){
    if(f->ref == 0){
      f->ref = 1;
      release(&ftable.lock);
      return f;
    }
  }
  release(&ftable.lock);
  return 0;
}

// Increment ref count for file f.
struct file*
filedup(struct file *f)
{
  acquire(&ftable.lock);
  if(f->ref < 1)
    panic("filedup");
  f->ref++;
  release(&ftable.lock);
  return f;
}

// Close file f.  (Decrement ref count, close when reaches 0.)
void
fileclose(struct file *f)
{
  struct file ff;

  acquire(&ftable.lock);
  if(f->ref < 1)
    panic("fileclose");
  if(--f->ref > 0){
    release(&ftable.lock);
    return;
  }
  ff = *f;
  f->ref = 0;
  f->type = FD_NONE;
  release(&ftable.lock);

  if(ff.type == FD_PIPE)
    pipeclose(ff.pipe, ff.writable);
  else if(ff.type == FD_INODE){
    begin_op();
    iput(ff.ip);
    end_op();
  }
}
```
可以看到一共三处，分别是分配文件结构、duplicate文件描述符、关闭文件描述符。很明显，这些都不在中断handler中，因此这里打开中断对正常处理是没问题的。

总的来说，能否打开中断需要细致分析临界区的代码有没有可能hold中断handler中acquire的锁，如果有的话，就不能打开中断，因为任何时候任何中断都有可能发生，如果没有，那就无所谓了。

