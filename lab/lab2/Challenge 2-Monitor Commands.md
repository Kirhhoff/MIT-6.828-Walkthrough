# Challenge 2: Monitor Commands
> **Extend the JOS kernel monitor with commands to:**
> 1. **Display in a useful and easy-to-read format all of the physical page mappings (or lack thereof) that apply to a particular range of virtual/linear addresses in the currently active address space. For example, you might enter 'showmappings 0x3000 0x5000' to display the physical page mappings and corresponding permission bits that apply to the pages at virtual addresses 0x3000, 0x4000, and 0x5000.**
> 2. **Explicitly set, clear, or change the permissions of any mapping in the current address space.** 
> 3. **Dump the contents of a range of memory given either a virtual or physical address range. Be sure the dump code behaves correctly when the range extends across page boundaries!**
> 4. **Do anything else that you think might be useful later for debugging the kernel. (There's a good chance it will be!)**

这个challenge是我最早做的，因为看起来更接近应用层，但是花的时间也是比较多的，主要是cli工具的参数检查、字符转换很烦，也没有stdlib，要自己写一点，但是总的来说这个不太需要查资料等等，倒是进度可期。

总的来说，仅实现了1，2，因为感觉3没啥用，也暂时没想到其他需要的debug工具。

对1，基本就是按照要求来的，对2，除了Present位设置为不可写，其他也都是按照要求来的。

1. smps
   ```
    int 
    mon_showmappings(int argc,char** argv,struct Trapframe* f){
        char hint[]="\nPlease pass arguments in correct formats, for example:\n"
                    "  smps 0x3000 0x5000 ---show the mapping from va=0x3000 to va=0x5000\n"
                    "  smps 0x3000 100 ---show the mapping of 100 virtual pages from va=0x3000\n"
                    "  smps 0x3000 ---show the mapping of va=0x3000 only\n";

        uintptr_t va_start;	
        uint32_t n_pages;
        validate_and_retrieve(argc,argv,&va_start,&n_pages,hint);

        // headline
        cprintf(
            "G: global   I: page table attribute index D: dirty\n"
            "A: accessed C: cache disable              T: write through\n"
            "U: user     W: writeable                  P: present\n"
            "---------------------------------\n"
            "virtual_ad  physica_ad  GIDACTUWP\n");

        uintptr_t va;
        int cnt;
        extern pde_t* kern_pgdir;

        // print out message w.r.t each page
        for(cnt=0;cnt<n_pages;cnt++){
            va=va_start+cnt*PGSIZE;
            if(P_PDE(kern_pgdir,va)&&P_PTE(kern_pgdir,va)){
                
                // transform permission to binary string
                char permission[10];
                permission[9]='\0';
                num2binstr(PERM(kern_pgdir,va),permission,9);

                // iff page is present
                cprintf("0x%08x  0x%08x  %s\n",va,PTE_ADDR(PTE(kern_pgdir,va)),permission);
                continue;
            }

            // once pde or pte is not present, print blank line
            cprintf("0x%08x  ----------  ---------\n",va);	
        }
            
        return 0;
    }
    ```
2. stp,clp
    ```
    int mon_setpermissions(int argc, char **argv, struct Trapframe *tf){
        char hint[]="\nPlease pass arguments in correct formats, for example:\n"
                    "  stp 0x3000 0x5000 AD ---set permission bit A and D from va=0x3000 to va=0x5000\n"
                    "  stp 0x3000 100 AD---set permission bit A and D of 100 virtual pages from va=0x3000\n"
                    "  stp 0x3000 AD---set permission bit A and D of va=0x3000 only\n"
                    "\n"
                    "G: global   I: page table attribute index D: dirty\n"
                    "A: accessed C: cache disable T: write through\n"
                    "U: user     W: writeable     P: present\n"
                    "\n"
                    "ps: P is forbbiden to set by hand\n";
        change_permissions(argc,argv,1,hint);
        cprintf("Permission has been updated:\n");
        mon_showmappings(argc-1,argv,tf);

        return 0;
    }

    int mon_clearpermissions(int argc, char **argv, struct Trapframe *tf){
        char hint[]="\nPlease pass arguments in correct formats, for example:\n"
                    "  clr 0x3000 0x5000 AD ---clear permission bit A and D from va=0x3000 to va=0x5000\n"
                    "  clr 0x3000 100 AD---clear permission bit A and D of 100 virtual pages from va=0x3000\n"
                    "  clr 0x3000 AD---clear permission bit A and D of va=0x3000 only\n"
                    "\n"
                    "G: global   I: page table attribute index D: dirty\n"
                    "A: accessed C: cache disable T: write through\n"
                    "U: user     W: writeable     P: present\n"
                    "\n"
                    "ps: P is forbbiden to clear by hand\n";
        change_permissions(argc,argv,0,hint);
        cprintf("Permission has been updated:\n");
        mon_showmappings(argc-1,argv,tf);

        return 0;
    }
    ```
其中用到的功能函数以及宏大致如下：
```
// macro used for string processing
#define IS_HEX(str) (((str[0])=='0')&&((str[1])=='x'))
#define HEX_VAL(str) (str2unum((str+2),16))
#define DEC_VAL(str) (str2unum(str,10))

#define START(str) (ROUNDDOWN(HEX_VAL(argv[1]),PGSIZE))

// nages=1 when there is only start address
// nages=n when there is a decimal second arg
// npages=(end-start)/PGSIZE when there is hexdecimal second arg
// also necessary round up/down
#define N_PAGES(argc,va_start,argv)((argc==2)?1:(IS_HEX(argv[2])?(ROUNDUP(HEX_VAL(argv[2]),PGSIZE)-va_start)/PGSIZE:DEC_VAL(argv[2])))

#define PDE(pgdir,va) (pgdir[(PDX(va))])
#define PTE_PTR(pgdir,va) (((pte_t*)KADDR(PTE_ADDR(PDE(pgdir,va))))+PTX(va))
#define PTE(pgdir,va) (*(PTE_PTR(pgdir,va)))

// macro to check present bit of pde/pte
#define P_PDE(pgdir,va) ((PDE(pgdir,va))&PTE_P)
#define P_PTE(pgdir,va) (PTE(pgdir,va)&PTE_P)
#define PERM(pgdir,va) (PTE(pgdir,va)-PTE_ADDR(PTE(pgdir,va)))
```
```
/***** Funtional inline tools for kernel monitor commands *****/

// transform a string to a number
// only support base 10 or 16
static inline int 
str2unum(char* str,int base){
	int ret=0,offset=0;
	if(base==10)
		while(*str&&(*str>='0')&&(*str<='9')){
			offset=(*str++)-'0';// char to number
			ret=base*ret+offset;
		}
	else if(base==16)
		while(*str&&(((*str>='0')&&(*str<='9'))||((*str>='a')&&(*str<='f')))){
			offset=*str-(*str<='9'?'0':('a'-10));// char to number
			ret=base*ret+offset;
			str++;
		}
	return *str?-1:ret;
}

// transfrom a number into 2-based string of
// specified bit
static inline void
num2binstr(uint32_t num,char* store,int bit){
	while(bit){
		store[--bit]=num%2+'0';
		num/=2;
	}
}

// map a capital character to 
// permission number
static inline uint32_t
char2perm(char c){
	switch (c)
	{
	case 'G': return PTE_G;
	case 'D': return PTE_D;
	case 'A': return PTE_A;
	case 'C': return PTE_PCD;
	case 'T': return PTE_PWT;
	case 'U': return PTE_U;
	case 'W': return PTE_W;
	case 'P': return PTE_P;
	default: return -1;
	}
}

// map a string of capital character
// to permission number, panic when
// illegal perm char passed
static inline uint32_t
str2perm(char* str){
	uint32_t perm=0;
	while(*str)
		perm|=char2perm(*str++);

	if(perm<0)
		panic("Wrong permission argument\n");
	
	// setting for 'Present' bit will be
	// automatically cancelled(forbidden) 
	perm&=(~char2perm('P'));

	return perm;
}

// validate input args and retrieve them
// if legitimate, otherwise panic
// requirements:
// 1. argc >=2
// 2. argv[1] written in hexdecimal form with 0x prefix
// 3. argv[2] written in hexdecimal or decimal form, the former
//    represents end page address, the latter represents number
//    of pages to specify
// 4. retrieved va_start and n_pages must be non-negative
static inline void validate_and_retrieve(int argc,char** argv,uintptr_t* va_start,uint32_t* n_pages,char* hint){
	// check in case of empty args
	if(argc<=1)
		panic(hint);

	// validate and retrieve start address and n_pages
	if(!IS_HEX(argv[1]))
		panic(hint);
	*va_start=START(argv[1]);	
	*n_pages=N_PAGES(argc,*va_start,argv);

	// check for illegal arg
	if (*va_start<0||*n_pages<0)
		panic(hint);
}

// universe funtioncal tool to set/clear page permissions,
// whose behaviour depends on op
void change_permissions(int argc, char **argv,bool op,char* hint){
	uintptr_t va_start;	
	uint32_t n_pages;
	validate_and_retrieve(argc-1,argv,&va_start,&n_pages,hint);
	
	// retrieve target permissions
	uint32_t perm=str2perm(argv[argc-1]);
	uintptr_t va;
	pte_t* va_pte;
	int cnt;
	extern pde_t* kern_pgdir;
	for(cnt=0;cnt<n_pages;cnt++){
		va=va_start+cnt*PGSIZE;
		//check pte present
		if(P_PDE(kern_pgdir,va)&&P_PTE(kern_pgdir,va)){
			va_pte=PTE_PTR(kern_pgdir,va);
			//set or clear
			*va_pte=op?(*va_pte|perm):(*va_pte&(~perm));
		}
	}
}
```
最后效果大致如下：
```
K> help
help - Display this list of commands
kerninfo - Display information about the kernel
trace - Print the stack trace
smps - Show physical pages mapped to specific virtual address area
stp - Set permissions of specific virtual pages
clp - Clear permissions of specific virtual pages
```
```
K> smps 0xefff0000 16
G: global   I: page table attribute index D: dirty
A: accessed C: cache disable              T: write through
U: user     W: writeable                  P: present
---------------------------------
virtual_ad  physica_ad  GIDACTUWP
0xefff0000  ----------  ---------
0xefff1000  ----------  ---------
0xefff2000  ----------  ---------
0xefff3000  ----------  ---------
0xefff4000  ----------  ---------
0xefff5000  ----------  ---------
0xefff6000  ----------  ---------
0xefff7000  ----------  ---------
0xefff8000  0x00111000  000000011
0xefff9000  0x00112000  000000011
0xefffa000  0x00113000  000000011
0xefffb000  0x00114000  000000011
0xefffc000  0x00115000  000000011
0xefffd000  0x00116000  000000011
0xefffe000  0x00117000  000000011
0xeffff000  0x00118000  000000011
```
这是内核栈附近的page信息

设置U
```
K> stp 0xefff8000 8 U
Permission has been updated:
G: global   I: page table attribute index D: dirty
A: accessed C: cache disable              T: write through
U: user     W: writeable                  P: present
---------------------------------
virtual_ad  physica_ad  GIDACTUWP
0xefff8000  0x00111000  000000111
0xefff9000  0x00112000  000000111
0xefffa000  0x00113000  000000111
0xefffb000  0x00114000  000000111
0xefffc000  0x00115000  000000111
0xefffd000  0x00116000  000000111
0xefffe000  0x00117000  000000111
0xeffff000  0x00118000  000000111
```
清除U
```
K> clp 0xefff8000 8 U
Permission has been updated:
G: global   I: page table attribute index D: dirty
A: accessed C: cache disable              T: write through
U: user     W: writeable                  P: present
---------------------------------
virtual_ad  physica_ad  GIDACTUWP
0xefff8000  0x00111000  000000011
0xefff9000  0x00112000  000000011
0xefffa000  0x00113000  000000011
0xefffb000  0x00114000  000000011
0xefffc000  0x00115000  000000011
0xefffd000  0x00116000  000000011
0xefffe000  0x00117000  000000011
0xeffff000  0x00118000  000000011
```

基本上就是这样了，但是还有一些缺陷，比如这个challenge是先于large page做的，应当考虑到large page 的显示，但是这里没有实现，如果后续有兴趣可以再改一下

