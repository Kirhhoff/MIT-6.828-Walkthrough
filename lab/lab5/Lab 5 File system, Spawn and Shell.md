# Lab 5: File system, Spawn and Shell

这一部分为JOS添加文件系统，但在此之前我们先看一下xv6对文件系统的实现设计，并穿插其与JOS的设计对比。

从最后实现的内容来看，xv6的更接近现实中os对文件系统的实现，即将它作为内核的一部分放在内核态实现，但JOS则选择微内核结构，将fs作为一个独立的具有较高权限的守护进程在用户态实现，个人感觉理解起来稍微困难一些（各种页面映射与拷贝会比较绕）

block
---
从低到上，先是block读/写。xv6在内核地址空间中开辟了一个block数组，作为磁盘块的存储，同时又做成双向链表用LRU控制顺序。block的大小出于方便设置为与sector相同，512B。JOS则不同，它的block与页面大小相同（4KB），是开辟在用户fs守护进程中的，而且是将其地址空间中连续的3GB(0x10000000-0xD0000000)全部用于存储block，但这里与xv6有显著的不同。JOS对block的管理利用了页面的管理，从而使得那3GB只有block数据，而没有block的管理信息，相较之下，xv6的block结构除了512B的扇区数据，还有block的管理信息（refcount，prev next指针等），因此可能会由对其引发cache的低效。那么JOS对block的管理信息存储在哪里呢？page结构中。也就是利用page的refcount来管理其block，而page本身也是有dirty，access等bit信息的，因此也很便利。

但是如何知道某个虚拟页对应的是哪个block号呢？page结构中可没有这个信息啊。JOS采取了非常naive的做法，也就是只支持3GB的磁盘大小（2333），因此页面相对起始0x10000000的offset就能决定它对应的blockno了。同样，磁盘上的文件系统也会维护一个bitmap来记录分配情况，fs进程将bitmap加载到内存，每次分配/释放时都立即将bitmap写回文件系统。

对block的读，xv6的方式是从头到尾遍历双向链表，如果找到这个block的缓存，就返回，否则从尾到头反向再次遍历双向链表，寻找free block，分配并将磁盘上的内容读取到block结构中再返回。而JOS由于固定位置和利用页面映射，读取相对简单许多，利用页错误，当fs进程执行读取时的拷贝时，如果发现pte为空则触发页错误，handler负责读取相应的扇区到fs进程的内存页面中，结束后正常执行。

inode
---
xv6采用磁盘inode(dinode)和内存inode两种结构，内核中维护一个inode数组来进行管理。就硬盘inode来说，因我们对文件内容的寻址实际上是通过inode中存放的blockno进行的，因此inode中应当有磁盘号数组；同时硬连接数也是必要的，用来决定什么时候inode可以释放；其他的还有文件大小，inode类型（文件还是目录）等等信息。

内存中的inode除了上述信息，还需要维护两方面的信息：
1. 运行时信息。包括锁、内存引用数、是否空闲（也即是是否这个内存inode中加载了一个磁盘inode）
2. 与磁盘inode的映射信息。包含的磁盘inode的设备号和它在磁盘上的编号。没错，每个文件系统都会有预留的块专门用来存储inode结构，通过编号以及superblock中记录的存储inode的块起始位置就可以找到磁盘上的inode数据了。

对于目录，xv6设定为它的内容为一系列的entry，每个entry维护文件名和该文件对应的inode(每个文件一个inode)(因此同一个文件在不同目录里可能有不同名字，对应不同的硬连接)。在superblock中（blockno=1，boot blockno=0)维护了文件系统大小、块数、inode数、log数、以及log块inode块bitmap块分别的起始块号。关于根目录，这里是设定为固定值inode=1，这样启动时系统会自动去读取inode=1将其作为根目录。

相较之下，JOS直接没有用inode，而是设置了File结构用来表示磁盘上的文件，其中像inode一样，有数据block的号，但是还包括文件名，同时，目录中包含的不再是entry，而是直接的一个个File结构，因此这里并不支持硬连接，同时为了简化这里也没有支持软连接。superblock仅仅维护块数、magic number以及根File（因为不支持inode）。没有inode相关信息，也没有log相关信息（不支持），同时根目录也是直接放在了superblock中，简化了非常多。

关于inode的读写，我们根据文件读写来分析。

对inode的读写就是建立在block的读写基础上了，

fd
---
关于dirlookup我们直接跳过了，这个比较trivial。对于文件描述符，xv6与JOS的实现差别也很大。

xv6在内核态维护一个打开的file数组，包含了所有进程打开的file，而fd的分配是每个进程独有的，每个进程维护一个file指针数组，作为其打开的文件，而fd则是file指针在数组中的index。那么可能就会产生一个疑问，既然所有进程能打开的文件数由内核的file数组大小决定，那么inode数组的大小又表示什么呢？毕竟一个file对应一个inode啊，他们的占用情况难道不应该相同吗？所以问题就变成了，除了打开文件占用inode，是否还有其他过程会占用inode？有，dirlookup。在路径查询时，每到达一层目录就必须获得当前目录的inode，这个过程不会占用file，但inode会被占用。

对于进程打开文件，xv6的流程如下：
1. dirlookup得到文件对应inode。其中涉及目录文件的读取以及inode的分配，而此时只有inode块的数据会从磁盘加载到内核block数组中
2. 内核file数组分配file结构。这里不会涉及磁盘操作
3. 从当前进程分配fd。这里不会涉及磁盘操作
4. 将inode信息以及权限、文件类型等信息attach(set)到file结构上
5. 返回fd，文件打开完成。

JOS对fd的实现与xv6差别挺大，他使用三个结构：File，Fd和OpenFile（这里的Fd是一个结构体不是数字）File是文件在磁盘上的形式，Fd是文件在用户空间的形式，OpenFile是fs守护进程维护的信息，每个OpenFile联系一个File和一个Fd，类似于xv6的内核空间file，也就是说OpenFile的个数决定了系统能够支持的打开文件数(对同一个文件的多次打开也分别计数)

JOS打开文件的流程如下：
1. 用户态分配Fd
2. 发送ipc(包含Fd)给fs守护进程
3. fs进程分配OpenFile
4. dirlookup得到文件对应的File。其中涉及目录文件的读取，注意这里File不需要分配，而是直接在父目录文件的内容中，只需要将父目录的文件内容读入后将指针指向父目录中该File的起始位置即可
5. OpenFile联系Fd与File

在这里，Fd的虽然是个结构体，但由于发送ipc时只能传递一个数字和一个页面，Fd的分配是分配一整个页面然后作为参数传递的。

对于文件读取，两者的实现也有许多区别。

xv6读取文件的流程如下：
1. 传递fd与buffer作为参数进行read系统调用
2. 内核根据该进程fd的position指针与要读取的字节数决定是否需要将下一个磁盘块读入，如果需要，则根据inode中下一个磁盘的磁盘号将磁盘块由磁盘读入内存
3. 内核将数据copy到buffer后返回用户态

而JOS要复杂一点，因为传递的buffer地址不在fs进程的内存映射中，如何访问到client进程的buffer是一个问题。JOS采用的措施是链接到fs库的每个进程都会有一个结构体变量Fsipc作为传递数据的“通道”：
1. 传递fd与buffer作为参数给库函数
2. 库函数发送ipc给fs进程，并将当前进程的Fsipc页面一同发送，ipc系统调用自动将该页面映射到fs进程的Fsipc地址
3. fs进程计算要读取的block号，并检查该block是否在内存中（利用页表），如果不在，则读取时触发页错误将该页面加载入内存
4. fs进程将数据copy到Fsipc所在页面，返回给clien进程
5. client进程库函数将数据由Fsipc所在页面copy到传递进来的buffer，返回

可以看出，JOS比xv6多了一次copy，而且为了解决不同进程间的内存映射问题，JOS需要预留出一个页面用于进程间的通信（类似UTEMP）

log
---
JOS并不支持日志与回滚，xv6对此有支持。xv6的log架设在block层之上，inode之下，每一次对涉及磁盘写的系统调用都必须用begin_op、end_op包围。而一个transaction可以包含多个系统调用的写请求，以此提升系统的吞吐量。总的来说，类似于文件记录的磁盘号，log也会预留一个blockno数组，每次调用了log包裹的写操作时都会把要写入的blockno加入到log数组中，在commit时一同写入。begin_op与end_op确保了在end_op时检查是否应当提交，begin_op能在提交时确保没有涉及磁盘写操作的系统调用能够开始。

总的来说，commit分为四个阶段：
1. 将要写入的block拷贝到预留的log块中
2. 在log系统中记录
3. 将log块中的内容拷贝到目标磁盘块中
4. 擦去log系统中的记录

很容易看出，这样就能够确保文件系统的一致性。

---
对xv6与JOS的实现进行了回顾后，我们再做exercise就会清晰很多。

> Exercise 1. i386_init identifies the file system environment by passing the type ENV_TYPE_FS to your environment creation function, env_create. Modify env_create in env.c, so that it gives the file system environment I/O privilege, but never gives that privilege to any other environment.
>
> Make sure you can start the file environment without causing a General Protection fault. You should pass the "fs i/o" test in make grade.

这里JOS对fs进程特别归为了一类，对这类进程需要执行in/out指令，因此创建进程时我们需要对它的eflag寄存器额外设置IOPL：
```
void
env_create(uint8_t *binary, enum EnvType type)
{
	// LAB 3: Your code here.

	// If this is the file server (type == ENV_TYPE_FS) give it I/O privileges.
	// LAB 5: Your code here.
	int errno;
	struct Env* e;

	lock_page();
	// allocate a new env
	if((errno=env_alloc(&e,0))<0){
		unlock_page();
		panic("panic %e",errno);
	}
	unlock_page();

	// set env type
	e->env_type=type;

	if(type==ENV_TYPE_FS)
		e->env_tf.tf_eflags|=FL_IOPL_3;

	// load initial code to 
	// user space of env
	load_icode(e,binary);
}
```

Question:
---
1. Do you have to do anything else to ensure that this I/O privilege setting is saved and restored properly when you subsequently switch from one environment to another? Why?

并不需要，因为进行权限级切换的时候硬件会自动修改和恢复eflag寄存器

> Exercise 2. Implement the bc_pgfault and flush_block functions in fs/bc.c. bc_pgfault is a page fault handler, just like the one your wrote in the previous lab for copy-on-write fork, except that its job is to load pages in from the disk in response to a page fault. When writing this, keep in mind that (1) addr may not be aligned to a block boundary and (2) ide_read operates in sectors, not blocks.
>
> The flush_block function should write a block out to disk if necessary. flush_block shouldn't do anything if the block isn't even in the block cache (that is, the page isn't mapped) or if it's not dirty. We will use the VM hardware to keep track of whether a disk block has been modified since it was last read from or written to disk. To see whether a block needs writing, we can just look to see if the PTE_D "dirty" bit is set in the uvpt entry. (The PTE_D bit is set by the processor in response to a write to that page; see 5.2.4.3 in chapter 5 of the 386 reference manual.) After writing the block to disk, flush_block should clear the PTE_D bit using sys_page_map.
>
> Use make grade to test your code. Your code should pass "check_bc", "check_super", and "check_bitmap".

这里我们就要实现核心功能，磁盘内容的读取->页错误处理函数，磁盘内容的写回->flush_back

这两个都比较简单，前者我们要做的就是分配页面和读取磁盘数据到内存页面，后者我们检查是否在内存中以及Dirty位，没问题后写回即可。
```
// Fault any disk block that is read in to memory by
// loading it from disk.
static void
bc_pgfault(struct UTrapframe *utf)
{
	void *addr = (void *) utf->utf_fault_va;
	uint32_t blockno = ((uint32_t)addr - DISKMAP) / BLKSIZE;
	int r;

	// Check that the fault was within the block cache region
	if (addr < (void*)DISKMAP || addr >= (void*)(DISKMAP + DISKSIZE))
		panic("page fault in FS: eip %08x, va %08x, err %04x",
		      utf->utf_eip, addr, utf->utf_err);

	// Sanity check the block number.
	if (super && blockno >= super->s_nblocks)
		panic("reading non-existent block %08x\n", blockno);

	// Allocate a page in the disk map region, read the contents
	// of the block from the disk into that page.
	// Hint: first round addr to page boundary. fs/ide.c has code to read
	// the disk.
	//
	// LAB 5: you code here:

	addr=ROUNDDOWN(addr,PGSIZE);

	if((r=sys_page_alloc(0,addr,PTE_SYSCALL)<0))
		panic("in bc_pgfault, sys_page_alloc: %e", r);

	if((r=ide_read(blockno*BLKSECTS,addr,BLKSECTS)<0))
		panic("in bc_pgfault, ide_read: %e", r);

	// Clear the dirty bit for the disk block page since we just read the
	// block from disk
	if ((r = sys_page_map(0, addr, 0, addr, uvpt[PGNUM(addr)] & PTE_SYSCALL)) < 0)
		panic("in bc_pgfault, sys_page_map: %e", r);

	// Check that the block we read was allocated. (exercise for
	// the reader: why do we do this *after* reading the block
	// in?)
	if (bitmap && block_is_free(blockno))
		panic("reading free block %08x\n", blockno);
}
```
可以看到除去sanity check外，就是三个动作，根据磁盘号分配对应页面，读取磁盘块到内存页，但是这里最后还要抹去dirty位，因为读取到内存页时会涉及pte的dirty位，但实际上此时我们还没进行写操作。
```
// Flush the contents of the block containing VA out to disk if
// necessary, then clear the PTE_D bit using sys_page_map.
// If the block is not in the block cache or is not dirty, does
// nothing.
// Hint: Use va_is_mapped, va_is_dirty, and ide_write.
// Hint: Use the PTE_SYSCALL constant when calling sys_page_map.
// Hint: Don't forget to round addr down.
void
flush_block(void *addr)
{
	uint32_t blockno = ((uint32_t)addr - DISKMAP) / BLKSIZE;

	if (addr < (void*)DISKMAP || addr >= (void*)(DISKMAP + DISKSIZE))
		panic("flush_block of bad va %08x", addr);

	// LAB 5: Your code here.

	addr=ROUNDDOWN(addr,PGSIZE);

	if(!va_is_mapped(addr)||!va_is_dirty(addr))
		return;
	
	if(ide_write(blockno*BLKSECTS,addr,BLKSECTS)<0)
		panic("flush_block of bad write block");

	if(sys_page_map(0,addr,0,addr,uvpt[PGNUM(addr)]&PTE_SYSCALL)<0)
		panic("flush_block of bad page map");
}
```
同样这里写回，也是先check，而后ide_write写回，最后抹去对应页面的dirty位。

这里可能会有人想到，会不会产生race condition呢？因为这里抹去dirty位和前面的行为都是非原子性的，会不会产生我们这里刚写回，又有别的地方修改了这个页面，但是我们又抹去了dirty位呢？不可能。因为这个页面是fs进程的，只有fs进程回访问，而我们这个os的架构决定了，一个进程（线程）只能同时在一个cpu上运行，因此从进程本身的角度，是不会由两个同时运行的fs进程修改这个页面的；那么会不会这个页面分配给别的进程后，别的进程对这个页面修改呢？也不会。因为我们开头分析过，JOS对文件系统的访问是全部delegate到fs进程的，也就是说，fs进程作为一个proxy，会负责（拦截）所有对文件系统的读写，因此别的进程不经fs进程无法对磁盘在内存页面的映像进行修改，也就不会引发race condition.

> Exercise 3. Use free_block as a model to implement alloc_block in fs/fs.c, which should find a free disk block in the bitmap, mark it used, and return the number of that block. When you allocate a block, you should immediately flush the changed bitmap block to disk with flush_block, to help file system consistency.
> 
> Use make grade to test your code. Your code should now pass "alloc_block".

这里我们需要实现在磁盘上分配一个块。注意这里分配完之后需要立即写回，xv6也是如此，只不过方式不太一样，是将bitmap所在块标记为dirty，在transaction提交时将其写回。

```
int
alloc_block(void)
{
	// The bitmap consists of one or more blocks.  A single bitmap block
	// contains the in-use bits for BLKBITSIZE blocks.  There are
	// super->s_nblocks blocks in the disk altogether.

	// LAB 5: Your code here.
	uint32_t i,bit=0,no=0;

	for(i=0;i<super->s_nblocks/32;i++)
		if(bitmap[i])
			while (bit<32){
				if(bitmap[i]&(1<<bit)){
					bitmap[i]&=(~(1<<bit));
					no=32*i+bit;
					flush_block(&bitmap[i]);
					return no;
				}
				else
					bit++;
			}

	return -E_NO_DISK;
}
```

> Exercise 4. Implement file_block_walk and file_get_block. file_block_walk maps from a block offset within a file to the pointer for that block in the struct File or the indirect block, very much like what pgdir_walk did for page tables. file_get_block goes one step further and maps to the actual disk block, allocating a new one if necessary.
>
> Use make grade to test your code. Your code should pass "file_open", "file_get_block", and "file_flush/file_truncated/file rewrite", and "testfile".

这一步要实现的是根据文件中的磁盘下标号获得对应的虚拟地址，这一个功能分为两步，首先获得文件中对应磁盘下标的地址（通过direct和indirect推算），然后检查是否分配了磁盘号，没有则分配之，然后指向对应的虚拟地址。
```
static int
file_block_walk(struct File *f, uint32_t filebno, uint32_t **ppdiskbno, bool alloc)
{
       // LAB 5: Your code here.
	if (filebno<NDIRECT){
		*ppdiskbno=&f->f_direct[filebno];
		return 0;
	}

	filebno-=NDIRECT;
	
	if(filebno<NINDIRECT){
		if (!f->f_indirect){
			if (alloc){
				if((f->f_indirect=alloc_block())<0){
					f->f_indirect=0;
					return -E_NO_DISK;
				}
			}else
				return -E_NOT_FOUND;
		}
		*ppdiskbno=(uint32_t*)(DISKMAP+f->f_indirect*BLKSIZE+filebno*sizeof(uint32_t));
		return 0;
	}

	return -E_INVAL;
}
int
file_get_block(struct File *f, uint32_t filebno, char **blk)
{
	// LAB 5: Your code here.
	int r;
	uint32_t *bnp,bva;
	if(file_block_walk(f,filebno,&bnp,1)<0)
		return -E_INVAL;
	
	if(!*bnp){
		if((r=alloc_block())<0)
			return -E_NO_DISK;
		*bnp=r;
	}

	*blk=(char*)(DISKMAP+*bnp*BLKSIZE);

	return 0;
}
```
> Exercise 5. Implement serve_read in fs/serv.c.
>
> serve_read's heavy lifting will be done by the already-implemented file_read in fs/fs.c (which, in turn, is just a bunch of calls to file_get_block). serve_read just has to provide the RPC interface for file reading. Look at the comments and code in serve_set_size to get a general idea of how the server functions should be structured.
> 
> Use make grade to test your code. Your code should pass "serve_open/file_stat/file_close" and "file_read" for a score of 70/150.

有了之前的基础后，我们就可以实现fs进程的读取服务了，这里冗杂的读取已经实现好了，我们只要实现逻辑即可，总体上来说就是根据传入的fileid查找到OpenFile然后根据其相关的Fd找到offset，从那里开始读取，并更新Fd的offset
```
// Read at most ipc->read.req_n bytes from the current seek position
// in ipc->read.req_fileid.  Return the bytes read from the file to
// the caller in ipc->readRet, then update the seek position.  Returns
// the number of bytes successfully read, or < 0 on error.
int
serve_read(envid_t envid, union Fsipc *ipc)
{
	struct Fsreq_read *req = &ipc->read;
	struct Fsret_read *ret = &ipc->readRet;

	if (debug)
		cprintf("serve_read %08x %08x %08x\n", envid, req->req_fileid, req->req_n);

	// Lab 5: Your code here:
	struct OpenFile* o;
	int r;
	if((r=openfile_lookup(0,req->req_fileid,&o))<0)
		return r;
	if((r=file_read(o->o_file,ret->ret_buf,req->req_n,o->o_fd->fd_offset))<0)
		return r;
	o->o_fd->fd_offset+=r;
	return r;
}
```
> Exercise 6. Implement serve_write in fs/serv.c and devfile_write in lib/file.c.
>
> Use make grade to test your code. Your code should pass "file_write", "file_read after file_write", "open", and "large file" for a score of 90/150.

有了read后，write就简单了，这里我们直接贴代码了：
```

```
> Exercise 7. spawn relies on the new syscall sys_env_set_trapframe to initialize the state of the newly created environment. Implement sys_env_set_trapframe in kern/syscall.c (don't forget to dispatch the new system call in syscall()).
> 
> Test your code by running the user/spawnhello program from kern/init.c, which will attempt to spawn /hello from the file system.
> 
> Use make grade to test your code.

到这我们就要实现类似exec的功能，这里虽然我们需要实现的比较少，但还是来仔细分析这里的代码。首先，我们还是从磁盘中读取文件，这里主要是读取elf头，读取的大小就是一个sector，包含的信息足矣。紧接着是fork一个子进程，当然，这时子进程还无法运行。接下来设置trapframe和栈，如何设置呢？首先是初始化栈，这里我们可以先分配一个临时页面，设置好内容后再映射到子进程的USTACK页位置。init_stack的内容比较冗杂，但是总体上思路比较明确，就是把启动参数先拷贝到栈上，从上到下，分布是参数字符串，参数字符串地址数组(argv)，argc，然后将esp设置到那里。接下来根据elf头的信息，将不同的segment从磁盘读取到子进程的对应内存页中（同时考虑对其等等），至此fd可以关闭，磁盘操作已经完成。剩下的就是copy库的页面，将trapframe设置到子进程中，标记其为runnable，就可以了。

设置trapframe:
```
static int
sys_env_set_trapframe(envid_t envid, struct Trapframe *tf)
{
	// LAB 5: Your code here.
	// Remember to check whether the user has supplied us with a good
	// address!

	if((uint32_t)tf>=UTOP-sizeof(*tf))
		return -E_INVAL;

	struct Env* e;
	if(envid2env(envid,&e,1)<0)
		return -E_BAD_ENV;
	
	e->env_tf=*tf;
	return 0;
}
```
比较简单
> Exercise 8. Change duppage in lib/fork.c to follow the new convention. If the page table entry has the PTE_SHARE bit set, just copy the mapping directly. (You should use PTE_SYSCALL, not 0xfff, to mask out the relevant bits from the page table entry. 0xfff picks up the accessed and dirty bits as well.)
>
> Likewise, implement copy_shared_pages in lib/spawn.c. It should loop through all page table entries in the current process (just like fork did), copying any page mappings that have the PTE_SHARE bit set into the child process.

而我们这里要实现的就是copy库的页面。这里用了一个新的位来表示库的页面
```
// Copy the mappings for shared pages into the child address space.
static int
copy_shared_pages(envid_t child)
{
	extern unsigned char end[];
	pte_t pte;
	int r;
	// LAB 5: Your code here.
	for(uint32_t addr=0;addr<(uint32_t)UTOP;addr+=PGSIZE){
		if(!(uvpd[PDX((void*)addr)]&PTE_P))
			continue;
		pte=uvpt[PGNUM((void*)addr)];
		if((pte&PTE_SHARE)
		&&(pte&PTE_P)
		&&(pte&PTE_U)
		&&((r=sys_page_map(0,(void*)addr,child,(void*)addr,(pte&PTE_SYSCALL)|PTE_SHARE))<0))
			return r;
	}

	return 0;
}
```
fork的由于我实现了内核态的fork，所以修改的是内核态的duppage:
```
static int
fork_duppage(struct Env* childenv, void* addr)
{
	// LAB 4: Your code here.
	pte_t* pte=pgdir_walk(curenv->env_pgdir,addr,0);
	int err;

	if (!pte)
		return 0;

	// duppage only if the page is present and belongs to user
	if((*pte&PTE_P)&&(*pte&PTE_U)){
		if(*pte&PTE_SHARE){
			if((err=fork_page_map(childenv,addr,addr,(*pte&PTE_SYSCALL)|PTE_SHARE))<0)
				return err;
		}else{
			if((*pte&PTE_W)||(*pte&PTE_COW)){
				if((err=fork_page_map(childenv,addr,addr,PTE_COW))<0)
					return err;
				if((*pte&PTE_W)&&!(*pte&PTE_COW)){
					if((err=fork_page_map(curenv,addr,addr,PTE_COW))<0)
						return err;
				}
			}else{
				if((err=fork_page_map(childenv,addr,addr,0))<0)
					return err;
			}
		}
	}	
	
	return 0;
}
```
> Exercise 9. In your kern/trap.c, call kbd_intr to handle trap IRQ_OFFSET+IRQ_KBD and serial_intr to handle trap IRQ_OFFSET+IRQ_SERIAL.

在这里，我们需要加入keyborad中断。先前我们对keyborad的监听都是在某个进程等待输入时不停地poll键盘设备的，这里我们要加入没有进程poll键盘设备时也能拿到输入的中断：
```
	// Handle keyboard and serial interrupts.
	// LAB 5: Your code here.

	if (tf->tf_trapno == IRQ_OFFSET + IRQ_KBD){
		kbd_intr();
		return;
	}

	if (tf->tf_trapno == IRQ_OFFSET + IRQ_SERIAL){
		serial_intr();
		return;
	}
```
> Exercise 10.
> 
> The shell doesn't support I/O redirection. It would be nice to run sh <script instead of having to type in all the commands in the script by hand, as you did above. Add I/O redirection for < to user/sh.c.
> 
> Test your implementation by typing sh <script into your shell
>
> Run make run-testshell to test your shell. testshell simply feeds the above commands (also found in fs/testshell.sh) into the shell and then checks that the output matches fs/testshell.key.

最后这里，我们只要在shell的那个函数里面补全即可，类似于之前的cmd处理：
```
			// LAB 5: Your code here.
			if((fd=open(t,O_RDONLY))<0){
				cprintf("open %s for read: %e", t, fd);
				exit();
			}

			if(fd!=0){
				dup(fd,0);
				close(fd);
			}
			break;
```
打开对应的文件(文件名在t中)，将其dup到标准输入流即可。
