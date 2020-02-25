# Challeng 4: Page Fragment Management
> **Challenge! Since our JOS kernel's memory management system only allocates and frees memory on page granularity, we do not have anything comparable to a general-purpose malloc/free facility that we can use within the kernel. This could be a problem if we want to support certain types of I/O devices that require physically contiguous buffers larger than 4KB in size, or if we want user-level environments, and not just the kernel, to be able to allocate and map 4MB superpages for maximum processor efficiency. (See the earlier challenge problem about PTE_PS.)**
> 
> **Generalize the kernel's memory allocation system to support pages of a variety of power-of-two allocation unit sizes from 4KB up to some reasonable maximum of your choice. Be sure you have some way to divide larger allocation units into smaller ones on demand, and to coalesce multiple small allocation units back into larger units when possible. Think about the issues that might arise in such a system.**

这应该是这三个challenge最有工作量的部分了，而且跟monitor command的不同，这个需要参考一些资料才得以实现（对我来说），相对来说花的功夫比较多，再加上线上开学等等，以及构造测试来确保代码正确性，用了一天多时间才搞定，但也有许多收获，算是挺棒的challenge了！

关于实现的思路方面主要是参考了《Understaning Linux Kernel》第八章关于buddy system的介绍，数据结构也是简化了的linux结构，具体的一些操作方法实现可能不太相同；而关于具体的实现技巧，比如generic C programming则是参考了《Professional Linux Kernel Architecture》的Appendix C.2.7的许多tricks（打开了新世界，C的泛型编程:>）

如果对buddy system（伙伴系统）不太了解可以参考[百度百科](https://baike.baidu.com/item/%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F/523696?fr=aladdin)

准备部分
---
首先是双向环连接链表结构以及关于链表的操作：
```
// generic C linked list
struct list_head{
	struct list_head *prev,*next;
};

static inline void init_list_head(struct list_head* list_head)
{
	list_head->next=list_head->prev=list_head;
}

static inline void __list_add(struct list_head *prev,struct list_head *next,struct list_head *new)
{
	prev->next=next->prev=new;
	new->prev=prev;
	new->next=next;	
};

static inline void __list_del(struct list_head *prev,struct list_head *next)
{
	prev->next=next;
	next->prev=prev;
};

static inline void list_add(struct list_head *new,struct list_head *head)
{	
	__list_add(head,head->next,new);
}

static inline void list_del(struct list_head *entry)
{
	__list_del(entry->prev,entry->next);
};

static inline void list_add_tail(struct list_head *new,struct list_head *head)
{
	__list_add(head->prev,head,new);
}

static inline bool list_empty(struct list_head *list_head)
{
	return list_head->next==list_head;
}
```
为了能够进行前后追踪，用于伙伴系统，我们给PageInfo加两个成员（可以看出这里的PageInfo是高度简化了的linux page结构体）:
```
struct PageInfo {
	// Next page on the free list.
	struct PageInfo *pp_link;

	// pp_ref is the count of pointers (usually in page table entries)
	// to this page, for pages allocated using page_alloc.
	// Pages allocated at boot time using pmap.c's
	// boot_alloc do not have valid reference count fields.

	uint16_t pp_ref;

	// information used to support page frame fragmentaion
	// management of buddy system
	// 0 if the page is allocated
	// otherwise indicates page frame number
	// in pow of 2
	int order;

	// generic structure
	// used to implement doubly linked
	// PageInfos
	struct list_head list_head; 
};
```
同时为了实现简易的buddy system，引入free_area结构：
```
// datastructure used for managing a specific
// size of page frame fragment, which
// works with all free page frame fragment
// of such size
struct free_area{

	// number of list node(rather than
	// page frame) in this area
	uint32_t nfree;

	// list_head of all nodes in this area
	// note: the index of Pageinfos pointed
	//      by these nodes are unordered
	struct list_head free_list_head;
};
```

关于泛型编程，我们引入这四个宏：
```
// Return the offset of 'member' relative to the beginning of a struct type
#define offsetof(type, member)  ((size_t) (&((type*)0)->member))

// obtain the container(struct) pointer from a member pointer
// defined for generic programming
#define container_of(ptr,type,member)({ \
	const typeof(((type*)0)->member)* __mem_type_ptr=(ptr); \
	((type*)(((char*)__mem_type_ptr)-offsetof(type,member)));\
})

// alias
#define list_entry(ptr,type,member) (container_of(ptr,type,member))
#define for_each_list(itr,list_head) \
	for(itr=(list_head)->next;itr!=list_head;itr=itr->next)
```
分别是:
1. 获取成员在结构体内的偏移量
2. 根据某成员指针及结构体类型，成员名称获取外部的容器（结构体）指针
3. 2的alias
4. 对链表进行for each循环操作

实现部分
---
malloc和free:
```
struct PageInfo*
malloc(uint32_t order){

	// check order
	if(order>MAX_ORDER)
		return NULL;
	
	// give priority to corresponding free_area
	if(!list_empty(&free_areas[order].free_list_head))
		return alloc_from_area(order,order);
	else{
		int forder=order;
		struct PageInfo* ret_pages;

		// search from larger area
		while (++forder<=MAX_ORDER){
			if(!list_empty(&free_areas[forder].free_list_head)){
				ret_pages=alloc_from_area(forder,order);

				// here fragments of different size turn out to be buddies
				set_buddy(ret_pages-pages+(1<<order),ret_pages-pages+(1<<forder),forder-1);
	
				return ret_pages;
			}
		}
	}

	return NULL;
}

void free(struct PageInfo* pp,int order){
	uint32_t pfn_idx=pp-pages,buddy_idx;
	uint32_t pfn_num=1<<order;
	struct PageInfo* buddy;

	while (order<MAX_ORDER){
		// obtain buddy information
		buddy_idx=pfn_idx^(1<<order);
		buddy=&pages[buddy_idx];

		// when the buddy is combinable
		// combine them and update necessary informaion
		// continue with order+1
		if(buddy->order==order&&buddy->pp_link){
			list_del(&buddy->list_head);
			free_areas[order].nfree--;
			buddy->order=0;
			pfn_idx&=buddy_idx;
			order++;
		}else{

			// order reaches maximum feasible combination
			list_add_tail(&pages[pfn_idx].list_head,&free_areas[order].free_list_head);
			free_areas[order].nfree++;
			return;
		}
	}

	// order reaches MAX_ORDER
	list_add_tail(&pages[pfn_idx].list_head,&free_areas[MAX_ORDER].free_list_head);
	free_areas[MAX_ORDER].nfree++;
}
```
相关的辅助性函数操作如下：
```
#define MAX_ORDER 10 		// log2(max continuous page frame number(1024) permitted to allocate)

struct free_area free_areas[MAX_ORDER+1]; // Free areas of 2^order page frames for malloc() and free()

// 0-initialize global free_areas array
// set it sooner
static inline void init_free_areas(){
	for(int order=0;order<=MAX_ORDER;order++){
		init_list_head(&free_areas[order].free_list_head);
		free_areas[order].nfree=0;
	}
}

// Used to set buddy information and update corresponding free_area
// [i_start, i_end) must be a free page number area
// meanwhile i_start-1 and i_end must have been allocated
void set_buddy(uint32_t i_start,uint32_t i_end,int order){

	// pfn number of this order
	uint32_t pfn_num=1<<order;

	// recursion stop condition
	if(order<0||i_end<=i_start)
		return;
	
	// rounding
	uint32_t ri_start=ROUNDUP(i_start,pfn_num),
		ri_end=ROUNDDOWN(i_end,pfn_num);
	
	// check rounding
	// We have rounded the start up and the end down,
	// of which the consequence is, somtimes ri_start
	// maybe larger than ri_end. But note that the initial
	// start is still smaller than the initial end
	// Under such circumstance, we just directly call the same
	// function with order-1 because we can't just abandon them
	if(ri_start>=ri_end)
		set_buddy(i_start,i_end,order-1);
	else{

		// set buddy information and update free_area	
		for(uint32_t pfn=ri_start;pfn<ri_end;pfn+=pfn_num){
			pages[pfn].order=order;
			list_add_tail(&pages[pfn].list_head,&free_areas[order].free_list_head);
			free_areas[order].nfree++;
		}
	
		// recursively set left and right rest regions
		set_buddy(i_start,ri_start,order-1);
		set_buddy(ri_end,i_end,order-1);
	}
}

// reset pageinfos of allocated pages
static void set_alloc_pageinfo(int i_start,int i_end){
	pages[i_start].order=0;
	for(int i=i_start;i<i_end;i++){
		pages[i].pp_link=NULL;
		pages[i].pp_ref++;
	}
}

// From an 'area_order' free_area allocate an 'alloc_order' block
// meanwhile update necessary information(except buddy information, with 
// which will be dealt later)
//
// note: necessary available free block check should be performed by caller
static struct PageInfo* alloc_from_area(int area_order,int alloc_order){
	struct list_head* head=&free_areas[area_order].free_list_head;

	// start of pageinfo of allocated block
	struct PageInfo* ret_pages=list_entry(head->next,struct PageInfo,list_head);

	// update free_area 
	list_del(head->next);
	free_areas[area_order].nfree--;

	// set pageinfos
	set_alloc_pageinfo(ret_pages-pages,ret_pages-pages+(1<<alloc_order));

	return ret_pages;
}
```
这里值得一提的是set_buddy采用了递归而不是linux里面的while，主要是这样自然点，但是关于边界情况，也就是函数中的一部分if条件都是debug后才认识到的，这个函数也是出问题最多的，因为buddy的信息确实相对难考虑一些

最后是验证函数；
```
// simple checks
static void check_malloc_and_free(){
	int cnt,order;
	
	struct PageInfo* pp;
	for(order=0;order<=6;order++){
		cnt=free_areas[order].nfree;
		pp=malloc(order);
		assert(free_areas[order].nfree==cnt-1);
		free(pp,order);
		assert(free_areas[order].nfree==cnt);
	}

	for(order=7;order<=9;order++){
		cnt=free_areas[MAX_ORDER].nfree;
		for(int _order=order;_order<=9;_order++)
			assert(free_areas[_order].nfree==0);
		pp=malloc(order);
		for(int _order=order;_order<=9;_order++)
			assert(free_areas[_order].nfree==1);
		assert(free_areas[MAX_ORDER].nfree==cnt-1);
		free(pp,order);
		for(int _order=order;_order<=9;_order++)
			assert(free_areas[_order].nfree==0);
		assert(free_areas[MAX_ORDER].nfree==cnt);
	}

	cprintf("check_malloc_and_free() succeeded!\n");
}
```
值得一提的是验证函数这里是在已知物理页的占用情况后据此编写的，因此可能需要根据不同的初始物理页分配情况来编写，在这里多说一点。

初始的物理页占用情况如下（根据之前的初始化情况可知）：
1. 0x0号页被BIOS信息占用
2. 0xa0-0xff号页被IO系统占用
3. 0x100-0x3ff号页被内核占用（这里用的是large page 4MB向上取整，因此直接从0x100分配到了0x3ff号页）
可以看出，从pfn=0x400开始全都是完整的4MB的block(order=10)，而前面的0x1-0x9f则慧禅师其他order的block，最终得到不同order的block如下：
```
order=0, nfree=1
order=1, nfree=1
order=2, nfree=1
order=3, nfree=1
order=4, nfree=1
order=5, nfree=2
order=6, nfree=1
order=7, nfree=0
order=8, nfree=0
order=9, nfree=0
order=10, nfree=31
```
其中order=0,1,2,3,4,5,6的block全都在0x1-0x9f这部分，而order=10的block全都在0x400往后的连续区域。因此设计了这个测试用例：
1. 对[0,6]的order分别malloc-free，malloc后对应order的nfree减-1，free后又加回来
2. 对[7,9]的order分别malloc，原则上从order一直到9应该都会+1，而order=10会-1，free后又复原
3. 对order=10 malloc，free测试


---
总的来说，这部分代码难写一些，要考虑合并等的情况，以及信息更新，还有不易察觉的边界，但是也还行，而且要注意测试用例的编写，不然写完了debug也很低效，实际上，这部分代码除了很少的一部分静态检查，主要就是靠这一个小测试用例来debug，而且从结果来看找出了非常多bug
```
Physical memory: 131072K available, base = 640K, extended = 130432K
check_page_free_list() succeeded!
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_free_list() succeeded!
check_page_installed_pgdir() succeeded!
check_malloc_and_free() succeeded!
```
