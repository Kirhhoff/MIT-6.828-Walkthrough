> Exercise 2. Use GDB's si (Step Instruction) command to trace into the ROM BIOS for a few more instructions, and try to guess what it might be doing. You might want to look at Phil Storrs I/O Ports Description, as well as other materials on the 6.828 reference materials page. No need to figure out all the details - just the general idea of what the BIOS is doing first.

# Answer
BIOS做了什么相较之下很好概括:

1. 上电自检，出现设备问题则蜂鸣或者直接关机（之前换内存条遇到过像内存没插好，直接啥都没显示自个儿关机了，主要这时候连内存都没，无任何能力为你显示信息）
2. 初始化实模式下的中断向量表，设置寄存器，对外围设备进行初始化
    - 关于中断向量表这里，可以参考《x86汇编语言：从实模式到保护模式》的第九章，大体上来说，除了必需的中断项由bios来初始化（外围设备的中断由自己的内存映射区域来提供，这句话后面会解释）外，其他中断项的入口地址均指向相同的地址，而那里只有一句话：ret，等后面内核加载进来以后自己按照需要去设置
    - 设置寄存器没啥好解释的
    - 初始化外围设备，包括对各个设备上的控制器进行初始化，因为开机的时候控制器处于未知的悬空状态 

2. 按照BIOS的设置从首选项，在这里是硬盘的0磁头0柱面1扇区，把它加载到0x7c00处，然后跳转到那里（也即是所谓的把控制权移交给bootloader）

# Postscript
这部分是记录我自己当时的一些疑惑

Q: BIOS是被谁、怎么加载到内存的？

A: 首先解释一下这个问题。从这部分实验以及以往os课程上，能大致理清BIOS，bootloader，内核之间的关系，从某种角度上来说算是级联加载，也就是先有bios上一段写死的代码来加载bootloader，再由bootloader开启保护模式，加载内核，然后内核负责后面的问题。所以很自然的一个问题是，谁来把BIOS加载到内存的？直接的答案是它不需要被加载到内存上。我们来看这张图：
![](https://img-blog.csdnimg.cn/20200218131051120.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg5MjI4OA==,size_16,color_FFFFFF,t_70)
要理解这个问题需要先明白8086当时的地址线和寄存器情况，也就是20根地址线但寄存器为了2的整幂只有16位。为啥非要搞20跟地址线呢？我想还是内存太小了，即便在那个年代，我想64K可访问地址也实在是太小了。有了20根地址线，可访问地址空间就上升到了1M，当然寻址就变成了段：偏移。但是这1MB的内存空间可不是全部都由内存提供的，在这里，只有那640KB是确实来自内存的，也就是说，只有当你访问这640KB的时候，才是真的在访问内存，而当你访问比如641K的地址时，看起来是在访问内存，其实是在访问这个设备，也就是说，640K-768K这段地址被映射到了VGA Display设备。BIOS也是这样，当你访问0xffff0时，其实是在访问BIOS芯片上写死的代码。这样一来，相当于BIOS天然就在内存上。那么当你有很大内存的时候，比如4GB，那么这960KB到1MB的内存是访问不到的，因为每当你访问这一块儿的时候，都映射到其他设备上去了，这也就是所谓的内存空洞的由来。但是可能又会产生一个疑问，开机的时候电脑咋知道该把哪些映射到其他设备上，哪些真正去访问内存呢？这个就是写死的了，作为计算机的标准，所以设定好就可以了。