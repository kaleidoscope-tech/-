# <center>操作系统lab2实验报告</center>
##### <center>小组成员：陈丽婷 彭伊朵 刘健琛 刘坤鹏</center>

### 分工
- 陈丽婷 1911539：练习二
- 彭伊朵 1911558：练习三
- 刘健琛 1911220：MELD，练习一
- 刘坤鹏 1911438：Challenge1，Challenge2

# 练习零：填写已有实验
```
本实验依赖实验1.请把你做的实验1的代码填入本实验中代码中有"LAB1"的注释相应部分。提示：可采用diff和patch工具进行半自动的合并（merge），也可用一些图形化的比较/merge工具来手动合并，比如meld，eclipse中的diff和merge工具，understand中的diff/merge工具等。
```
![](https://i.loli.net/2021/05/09/ru3ekfHnLcAdhIS.png)

在这样的界面上选择两个需要比较的文件打开



![image-20210419220539994](https://i.loli.net/2021/04/19/a69EAufjm2ZgCs4.png)

使用meld来进行对比合并。


# 练习一 实现first-fit连续物理内存分配算法（需要编程）

```
在实现first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示：在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改default_pmm.c中的default_init,default_init_memmap,default_alloc_pages,default_free_pages等相关函数。请仔细查看和理解default_pmm.c中的注释。
请在实验报告中简要说明你的设计实现过程。请回答如下问题：
你的first fit算法是否有进一步的改进空间
```

## 设计实现过程

first-fit 的基本思想：要求空闲区按地址递增的次序排列。当进行内存分配时，从空闲区表头开始顺序查找，直到找到第一个能滿足其大小要求的空闲区为止。分一块给请求者，余下部分仍留在空闲区中。

在实验代码中使用链表来进行物理内存分配，按照空闲页块起始地址来排序，形成一个有序的链表。

`free_list`用于记录空闲的内存块。 `nr_free`是空闲内存块的总数。

### 在`pmm.c` 中可以找到结构体代码如下：
```c
struct list_entry {
    struct list_entry *prev, *next; 
};

typedef struct list_entry list_entry_t; 
typedef struct {
    list_entry_t free_list;         
    unsigned int nr_free;   
} free_area_t; 
```

### 页表的定义如下：
```c
struct Page {
    int ref;       //映射此物理页的虚拟页的个数       
    uint32_t flags; //物理页属性    
    unsigned int property;  //连续空页有多少(只在地址最低页有值)
    list_entry_t page_link; // 双向链接各个Page结构的page_link
};
```

以上就是分配内存的各项结构体定义。

### 初始化内存分配映射的函数：
```c
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;//标志位清零
        SetPageProperty(base);//设置为保留页
        set_page_ref(p, 0);//将引用清零
		list_add_before(&free_list,&(p->page_link));
    }
    base->property = n;
    nr_free += n;
}
```

该函数可以实现n个块的初始化。

### 改写分配页表的函数：
```c
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    list_entry_t *le, *len;
    le = &free_list;

    while((le=list_next(le)) != &free_list) {
      struct Page *p = le2page(le, page_link);
      /*当有n个以上连续空闲页的时候开始分配
        将每一个链表指向的页表转换成页表。
        将这个链表指向从空闲链表中删除，实现页的分配。
        */
      if(p->property >= n){  
        int i;
        for(i=0;i<n;i++){
          len = list_next(le);
          struct Page *pp = le2page(le, page_link);
          SetPageReserved(pp);
          ClearPageProperty(pp);
          list_del(le);
          le = len;
        }
        if(p->property>n){
          (le2page(le,page_link))->property = p->property - n;//如果页块大小大于所需大小，分割页块
        }
        ClearPageProperty(p);
        SetPageReserved(p);
        nr_free -= n;
        return p;
      }
    }
    return NULL;
}
```

通过page的property属性判断空闲页的大小是否大于所需的页块大小。遍历整个空闲链表去寻找合适的空闲页。如果找到合适的空闲页，则重新设置标志位。然后从空闲链表中删除此页。如果当前空闲页的大小大于所需大小。则分割页块。如果合适则不进行操作，最后计算剩余空闲页个数并返回分配的页块地。

### 释放页表

```c
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);  
    assert(PageReserved(base));    //检查需要释放的页块是否已经被分配
    list_entry_t *le = &free_list; 
    struct Page * p;
    while((le=list_next(le)) != &free_list) {    //寻找合适的位置
      p = le2page(le, page_link); //获取链表对应的Page
      if(p>base){    
        break;
      }
    }
    for(p=base;p<base+n;p++){              
      list_add_before(le, &(p->page_link)); //将每一空闲块对应的链表插入空闲链表中
    }
    base->flags = 0;         //修改标志位
    set_page_ref(base, 0);    //引用清零
    ClearPageProperty(base);
    SetPageProperty(base);
    base->property = n;      //设置连续大小为n
    //如果是高位，则向高地址合并
    p = le2page(le,page_link) ;
    if( base+n == p ){
      base->property += p->property;
      p->property = 0;
    }
     //如果是低位且在范围内，则向低地址合并
    le = list_prev(&(base->page_link));
    p = le2page(le, page_link);
    if(le!=&free_list && p==base-1){ //满足条件，未分配则合并
      while(le!=&free_list){
        if(p->property){ 
          p->property += base->property;//因为是连续的，增加连续空闲页的值
          base->property = 0;
          break;
        }
        le = list_prev(le);
        p = le2page(le,page_link);
      }
    }

    nr_free += n;
    return ;
} 
```
这个函数的作用是释放已经使用完的页，把他们合并到`freelist`中。在`freelist`中查找合适的位置以供插入。改变被释放页的标志位，以及头部的计数器尝试在`freelist`中向高地址或低地址合并。

## 你的first fit算法是否有进一步的改进空间
可以用树来管理空闲页块 搜索的时间复杂度为 O(logn)

# 练习二：实现寻找虚拟地址对应的页表项（需要编程）
```
通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的get_pte函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。本练习需要补全get_pte函数 in kern/mm/pmm.c，实现其功能。请仔细查看和理解get_pte函数中的注释。
```
get_pte函数的调用关系图如下：
![](https://i.loli.net/2021/05/09/5G4bpZCcg1MKQli.png)
```
请在实验报告中简要说明你的设计实现过程。请回答如下问题：
1、请描述页目录项（Page Director Entry)和页表（Page Table Entry)中每个组成部分的含义以及对ucore而言的潜在用处。
2、如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
```
先参考代码注释：
```C
/* LAB2 EXERCISE 2: YOUR CODE
     *
     * IF you need to visit a physical address,please use KADDR()
     * please read pmm.h for useful macros
     *
     *Maybe you want help comment,BELOW comments can help you finish the code
     * 
     *Some Useful MACROs and DEFINEs,you can use them in below implementation.
     * MACROs for Functions:
     *   PDX(la)=the index of page directory entry of VIRTUAL ADDRESS la.
     *   KADDR(pa) : takes a physical address and returns the corresponding kernel virtual address.
     *   set_page_ref(page,1):means the page be referenced by one time
     *   page2pa(page): get the physical address of memory which this(struct Page *)page manages
     *   Struct Page * alloc_page() :allocation a page
     *   memset(void *s,char c,size_t n): Set the first n bytes of the memory area pointed by s
     *
     * DEFINEs:               to the specified value c
     *   PTE_P        0x001      //page table/directory entry flags bit: Present
     *   PTE_W        0x002      //page teble/directory entry flags bit:Writeable
     *   PTE_U        0x004      //page table/directory entry flags bit:User can access
     */
```
```C
#if 0
    pde_t *pdep = NULL;   //(1)find page directory entry
    if(0){                //(2)check if entry is not present
                          //(3)check if creating is needed,then alloc page for page table
                          //CAUTION: this page is used for page table,not for common data page
                          //(4)set page reference
       uintptr_t pa=0;    //(5)get linear address of page
                          //(6)clear page content using memset
                          //(7)set page directory entry's permission
    }
    return NULL           //(8)return page table entry
#endif              
```
可以看到实现寻找虚拟地址对应的页表项大概需要这些步骤：
（1）找到页目录项（page directory entry）
（2）确认其present位是否为1
（3）确认是否需要为其创建新的页表
（4）设置页引用计数
（5）获取页的线性地址
（6）用memset将物理页初始化为0
（7）设置页表目录项的权限
（8）返回页表

根据已有注释，可以得出get_pte函数的代码如下：
```C
pte_t *
get_pte(pde_t *pgdir,uintptr_t la,bool create){
    /* LAB2 EXERCISE 2: YOUR CODE
     *
     * IF you need to visit a physical address,please use KADDR()
     * please read pmm.h for useful macros
     *
     *Maybe you want help comment,BELOW comments can help you finish the code
     * 
     *Some Useful MACROs and DEFINEs,you can use them in below implementation.
     * MACROs for Functions:
     *   PDX(la)=the index of page directory entry of VIRTUAL ADDRESS la.
     *   KADDR(pa) : takes a physical address and returns the corresponding kernel virtual address.
     *   set_page_ref(page,1):means the page be referenced by one time
     *   page2pa(page): get the physical address of memory which this(struct Page *)page manages
     *   Struct Page * alloc_page() :allocation a page
     *   memset(void *s,char c,size_t n): Set the first n bytes of the memory area pointed by s
     *
     * DEFINEs:               to the specified value c
     *   PTE_P        0x001      //page table/directory entry flags bit: Present
     *   PTE_W        0x002      //page teble/directory entry flags bit:Writeable
     *   PTE_U        0x004      //page table/directory entry flags bit:User can access
     */
#if 0
    pde_t *pdep = NULL;   //(1)find page directory entry
    if(0){                //(2)check if entry is not present
                          //(3)check if creating is needed,then alloc page for page table
                          //CAUTION: this page is used for page table,not for common data page
                          //(4)set page reference
       uintptr_t pa=0;    //(5)get linear address of page
                          //(6)clear page content using memset
                          //(7)set page directory entry's permission
    }
    return NULL           //(8)return page table entry
#endif              
    pde_t *pdep= &pgdir[PDX(la)]; //通过PDX(la)可以获得对应的PDE的索引，之后在pkdir中根据索引找到对应的PDE指针,从而获取PTE

    if(!(*pdep & PTE_P)){ //判断PDE的present位是否为1，若不为1就需要为其创建新的页表（PTT）
      struct Page *temPage;
      if(!create || (temPage = alloc_page()) == NULL){   //若创建失败则返回NULL
        return NULL;
      }
      set_page_ref(temPage,1);//用该函数进行物理页引用计数的更新
      uintptr_t pa = page2pa(temPage);
      memset(KADDR(pa),0,PGSIZE);//利用KADDR（）函数转化为内核虚拟地址，将temPage的物理页全部初始化为0
      *pdep = pa | PTE_U | PTE_W | PTE_P;
    }     
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)]; //返回对应的PTE
}
```

其中遇到的一些变量定义如下：
![](https://i.loli.net/2021/05/09/NtSTXm27JDZjOhs.png)
其中前10位用于页目录的索引，中间的10位用于页表的索引，最后十二位用于页偏移量

### 1、请描述页目录项（Page Director Entry)和页表（Page Table Entry)中每个组成部分的含义以及对ucore而言的潜在用处。

页目录项是指向储存页表的页面的, 所以本质上与页表项相同, 结构也应该差不多相同.主要就是第6位，PDE忽略，而PTE是判断是否有写入的脏位等。两者都是4B大小，其中高20bit被用于保存索引，低12bit用于保存页偏移量，可以通过在mmu.h中的一组宏定义发现（上述截图中可找到）。


### 2、如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

1、CPU将发生异常的线性地址la保存在CR2寄存器中
2、将寄存器中的值压入栈中
3、根据IDT表查询到对应的访问异常，跳转交给软件处理

# 练习三：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）
```
当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做 相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系 的二级页表项清除。请仔细查看和理解page_remove_pte函数中的注释。为此，需要补全在 kern/mm/pmm.c中的page_remove_pte函数。
```
page_remove_pte函数的调用关系图如下所示：

![](https://www.hualigs.cn/image/60976c71308ea.jpg)

```
图2 page_remove_pte函数的调用关系图 
请在实验报告中简要说明你的设计实现过程。请回答如下问题：
数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？
如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 
鼓励通过编程来 具体完成这个问题
```

仔细查看和理解page_remove_pte函数中的注释
```c++
//page_remove_pte - free an Page sturct which is related linear address la
//                - and clean(invalidate) pte which is related linear address la
//note: PT is changed, so the TLB need to be invalidate 

static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
    /* LAB2 EXERCISE 3: YOUR CODE
     *
     * Please check if ptep is valid, and tlb must be manually updated if mapping is updated
     *
     * Maybe you want help comment, BELOW comments can help you finish the code
     *
     * Some Useful MACROs and DEFINEs, you can use them in below implementation.
     * MACROs or Functions:
     *   struct Page *page pte2page(*ptep): get the according page from the value of a ptep
     *   free_page : free a page
     *   page_ref_dec(page) : decrease page->ref. NOTICE: ff page->ref == 0 , then this page should be free.
     *   tlb_invalidate(pde_t *pgdir, uintptr_t la) : Invalidate a TLB entry, but only if the page tables being
     *                        edited are the ones currently in use by the processor.
     * DEFINEs:
     *   PTE_P           0x001                   // page table/directory entry flags bit : Present
     */
#if 0
    if (0) {                      //(1) check if this page table entry is present
        struct Page *page = NULL; //(2) find corresponding page to pte
                                  //(3) decrease page reference
                                  //(4) and free this page when page reference reachs 0
                                  //(5) clear second page table entry
                                  //(6) flush tlb
    }
#endif
}

```

其中可以看到：
```
     *   struct Page *page pte2page(*ptep):从一个ptep的值中获取相应的页面
     *   free_page : 释放页面
     *   page_ref_dec(page) : decrease page->ref. NOTICE: ff page->ref == 0 , then this page should be free.
     *   tlb_invalidate(pde_t *pgdir, uintptr_t la) : 使TLB条目无效，但仅当页面表存在
```

释放虚地址所在的页和取消对应二级页表项的映射可以大概分为如下步骤：

1. 通过PTE_P判断该ptep是否存在；
2. 判断Page的ref的值是否为0，若为0，则说明此时没有任何逻辑地址被映射到此物理地址，换句话说当前物理页已没人使用，因此调用free_page函数回收此物理页，使得该物理页空闲；若不为0，则说明此时仍有至少一个逻辑地址被映射到此物理地址，因此不需回收此物理页；
3. 把表示虚地址与物理地址对应关系的二级页表项清除；
4. 更新TLB；

基于以上可以编写代码：
```c++
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {

if (*ptep & PTE_P) {
        // 如果对应的二级页表项存在，如果*ptep存在，则其与PTE_P相与，得到的是1；
        struct Page *page = pte2page(*ptep);  // 获得*ptep对应的Page结构
        // 关联的page引用数自减1，page_ref_dec的返回值是现在的page->ref
       if(page_ref_dec(page) == 0) {
            // 如果自减1后，引用数为0，需要free释放掉该物理页
            free_page(page);
        }
        // 把表示虚地址与物理地址对应关系的二级页表项清除(通过把整体设置为0)
        *ptep = 0;
        // 由于页表项发生了改变，需要使TLB快表无效
        tlb_invalidate(pgdir, la);
    }
   
}
```


### 1、数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

![](https://www.hualigs.cn/image/6097a202c613e.jpg)
![](https://www.hualigs.cn/image/6097a1c9ab515.jpg)
A：所有的物理页都有一个描述它的Page结构，所有的页表都是通过alloc_page()分配的，每个页表项都存放在一个Page结构描述的物理页中；如果 PTE 指向某物理页，同时也有一个Page结构描述这个物理页。所以有两种对应关系：

(1)可以通过 PTE 的地址计算其所在的页表的Page结构：

将虚拟地址向下对齐到页大小，换算成物理地址(减 KERNBASE), 再将其右移 PGSHIFT(12)位获得在pages数组中的索引PPN，&pages[PPN]就是所求的Page结构地址。


### 2、如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？

A：此时虚拟地址和物理地址的映射关系是：phy addr + KERNBASE = virtual addr

所以如果想让虚拟地址=物理地址，则只要让KERNBASE = 0，因此去修改memlayout.h的宏定义
```
#define KERNBASE            0xC0000000
```
把KERNBASE 改为 0

# Challenge1：buddy system(伙伴系统)分配算法(需要编程)

<blockquote>Buddy System算法把系统中的可用存储空间划分为存储块(Block)来进行管理, 每个存储块的 大小必须是2的n次幂(Pow(2, n)), 即1, 2, 4, 8, 16, 32, 64, 128…

参考伙伴分配器的一个极简实现， 在ucore中实现buddy system分配算法，要求有比较充分的测试用例说明实现的正确性，需要有设计文档。</blockquote>

查阅资料可知，Buddy system本质上也是一种连续物理内存的分配算法，主要采用二叉树结构来实现连续内存的分配；从功能上讲等价于练习一中的first-fit算法，可以进行替换。
具体原理可以参考下图：
![](https://i.loli.net/2021/05/07/ZlIaTs4jgBxJbcE.png)

网上对buddy system的算法定义如下：

##### 分配内存：

```
#内存分配伪代码
分配内存()｛
	if（存在合适的内存块）｛
		分配给应用程序；
	｝else｛
		将相对最合适的块均分生成两个块；
     	        分配内存()；
		｝
}
```

当内存块大小大于等于所需内存块大小并且最接近2的幂，比如需要27，实际分配32，这样的内存块可以叫做合适的内存块，而且符合大小为2的n次幂结构。

##### 释放内存

释放内存块可以看作是分配内存块的逆过程。
从最底层寻找要释放的块，查看相邻块是否释放，如果没有释放，那么将这两个块给合并，直到不能再合并为止，递归重复此过程即可实现内存块的释放。  
buddy system采用二叉树来实现此功能，这里我们才用数组来模拟二叉树进行操作，采用数组的下标索引加偏移的方式来实现。大致形成效果如下图，高层节点对应于大块，低层节点对应于小块，节点属性用来标记。  

![](https://i.loli.net/2021/05/07/x1NOZTbdeCGsDAw.png)

##### 代码实现

仿照default_pmm的格式。

1.首先，编写buddy.h（仿照default_pmm.h），将pmm_manager修改为buddy_pmm_manager

```
#ifndef __KERN_MM_BUDDY_PMM_H__
#define  __KERN_MM_BUDDY_PMM_H__

#include <pmm.h>

extern const struct pmm_manager buddy_pmm_manager;

#endif /* ! __KERN_MM_DEFAULT_PMM_H__ */
```
然后，进入buddy.c文件
索引机制：每层的左子树的下标为2^层-1，根节点为0，所以如果我们得到[index]节点的所在层数level，那么偏移量offset的计算可以归结为  
<center>(index-2^level+1) * node_size = (index+1)node_size – node_size2^level。</center>  
其中size的计算为
<center>2^(max_depth-level)</center>所以<center>node_size * 2^level = 2^max_depth = size</center>
综上所述，可得公式
<center>offset=(index+1)*node_size – size</center>

PS：式中索引的下标均从0开始，size为内存总大小，node_size为内存块对应大小。

由上，完成宏定义。

```
#define LEFT_LEAF(index) ((index) * 2 + 1)
#define RIGHT_LEAF(index) ((index) * 2 + 2)
#define PARENT(index) ( ((index) + 1) / 2 - 1)
```

2.构造新的块大小结构来实现快速查找块大小以满足2的n次幂规定。

```
static unsigned fixsize(unsigned size) {
  size |= size >> 1;
  size |= size >> 2;
  size |= size >> 4;
  size |= size >> 8;
  size |= size >> 16;
  return size+1;
}
```

构造buddy system最基本的数据结构，并初始化一个用来存放“二叉树”的数组。

```
struct buddy {
  unsigned size;//表明管理内存
  unsigned longest; 
};
struct buddy root[10000];//存放二叉树的数组，用于内存分配
```

buddy system是需要和实际指向空闲块双链表配合使用的，所以需要先各自初始化数组和指向空闲块的双链表。

```
//先初始化双链表
free_area_t free_area;
#define free_list (free_area.free_list)
#define nr_free (free_area.nr_free)

static void buddy_init()
{
    list_init(&free_list);
    nr_free=0;
}

//再初始化buddy system的数组
void buddy_new( int size ) {
  unsigned node_size;    //传入的size是这个buddy system表示的总空闲空间；node_size是对应节点所表示的空闲空间的块数
  int i;
  nr_block=0;
  if (size < 1 || !IS_POWER_OF_2(size))//是否符合条件，不符合则返回
    return;

  root[0].size = size;     //设置根节点
  node_size = size * 2;   //认为总结点数是size*2

  for (i = 0; i < 2 * size - 1; ++i) {
    if (IS_POWER_OF_2(i+1))    //如果i+1是2的倍数，那么该节点所表示的二叉树就要到下一层了
      node_size /= 2;
    root[i].longest = node_size;   //longest是该节点所表示的初始空闲空间块数
  }
  return;
}
```

根据pmm.h里面对于pmm_manager的统一结构化定义，我们需要对buddy system完成如下函数：

```
const struct pmm_manager buddy_pmm_manager = {
    .name = "buddy_pmm_manager",      // 管理器的名称
    .init = buddy_init,               // 初始化管理器
    .init_memmap = buddy_init_memmap, // 设置可管理的内存,初始化可分配的物理内存空间
    .alloc_pages = buddy_alloc_pages, // 分配>=N个连续物理页,返回分配块首地址指针 
    .free_pages = buddy_free_pages,   // 释放包括自Base基址在内的，起始的>=N个连续物理内存页
    .nr_free_pages = buddy_nr_free_pages, // 返回全局的空闲物理页数量
    .check = buddy_check,             //举例检测这个pmm_manager的正确性
};
```

- 初始化管理器（这个已在上面完成）

- 初始化可管理的物理内存空间

```
static void
buddy_init_memmap(struct Page *base, size_t n)
{
    assert(n>0);
    struct Page* p=base;
    for(;p!=base + n;p++)
    {
        assert(PageReserved(p));
        p->flags = 0;
        p->property = 1;
        set_page_ref(p, 0);    //表明空闲可用
        SetPageProperty(p);
        list_add_before(&free_list,&(p->page_link));     //向双链表中加入页的管理部分
    }
    nr_free += n;     //表示总共可用的空闲页数
    int allocpages=UINT32_ROUND_DOWN(n);
    buddy2_new(allocpages);    //传入所需要表示的总内存页大小，让buddy system的数组得以初始化
}
```

- 分配所需的物理页，返回分配块首地址指针

```
//分配的逻辑是：首先在buddy的“二叉树”结构中找到应该分配的物理页在整个实际双向链表中的位置，而后把相应的page进行标识表明该物理页已经分出去了。
static struct Page*
buddy_alloc_pages(size_t n){
  assert(n>0);
  if(n>nr_free)
   return NULL;
  struct Page* page=NULL;
  struct Page* p;
  list_entry_t *le=&free_list,*len;
  rec[nr_block].offset=buddy2_alloc(root,n);//记录偏移量
  int i;
  for(i=0;i<rec[nr_block].offset+1;i++)
    le=list_next(le);
  page=le2page(le,page_link);
  int allocpages;
  if(!IS_POWER_OF_2(n))
   allocpages=fixsize(n);
  else
  {
     allocpages=n;
  }
  //根据需求n得到块大小
  rec[nr_block].base=page;//记录分配块首页
  rec[nr_block].nr=allocpages;//记录分配的页数
  nr_block++;
  for(i=0;i<allocpages;i++)
  {
    len=list_next(le);
    p=le2page(le,page_link);
    ClearPageProperty(p);
    le=len;
  }//修改每一页的状态
  nr_free-=allocpages;//减去已被分配的页数
  page->property=n;
  return page;
}


//以下是在上面的分配物理内存函数中用到的结构和辅助函数
struct allocRecord//记录分配块的信息
{
  struct Page* base;
  int offset;
  size_t nr;//块大小，即包含了多少页
};

struct allocRecord rec[80000];//存放偏移量的数组
int nr_block;//已分配的块数

int buddy2_alloc(struct buddy2* self, int size) { //size就是这次要分配的物理页大小
  unsigned index = 0;  //节点的标号
  unsigned node_size;  //用于后续循环寻找合适的节点
  unsigned offset = 0;

  if (self==NULL)//无法分配
    return -1;

  if (size <= 0)//分配不合理
    size = 1;
  else if (!IS_POWER_OF_2(size))//不为2的幂时，取比size更大的2的n次幂
    size = fixsize(size);

  if (self[index].longest < size)//根据根节点的longest，发现可分配内存不足，也返回
    return -1;

  //从根节点开始，向下寻找左右子树里面找到最合适的节点
  for(node_size = self->size; node_size != size; node_size /= 2 ) {
    if (self[LEFT_LEAF(index)].longest >= size)
    {
       if(self[RIGHT_LEAF(index)].longest>=size)
        {
           index=self[LEFT_LEAF(index)].longest <= self[RIGHT_LEAF(index)].longest? LEFT_LEAF(index):RIGHT_LEAF(index);
         //找到两个相符合的节点中内存较小的结点
        }
       else
       {
         index=LEFT_LEAF(index);
       }  
    }
    else
      index = RIGHT_LEAF(index);
  }

  self[index].longest = 0;//标记节点为已使用
  offset = (index + 1) * node_size - self->size;  //offset得到的是该物理页在双向链表中距离“根节点”的偏移
  //这个节点被标记使用后，要层层向上回溯，改变父节点的longest值
  while (index) {
    index = PARENT(index);
    self[index].longest = 
      MAX(self[LEFT_LEAF(index)].longest, self[RIGHT_LEAF(index)].longest);
  }
  return offset;
}
```

- 释放指定的内存页大小

```
void buddy_free_pages(struct Page* base, size_t n) {
  unsigned node_size, index = 0;
  unsigned left_longest, right_longest;
  struct buddy2* self=root;
  
  list_entry_t *le=list_next(&free_list);
  int i=0;
  for(i=0;i<nr_block;i++)  //nr_block是已分配的块数
  {
    if(rec[i].base==base)  
     break;
  }
  int offset=rec[i].offset;
  int pos=i;//暂存i
  i=0;
  while(i<offset)
  {
    le=list_next(le);
    i++;     //根据该分配块的记录信息，可以找到双链表中对应的page
  }
  int allocpages;
  if(!IS_POWER_OF_2(n))
   allocpages=fixsize(n);
  else
  {
     allocpages=n;
  }
  assert(self && offset >= 0 && offset < self->size);//是否合法
  nr_free+=allocpages;//更新空闲页的数量
  struct Page* p;
  for(i=0;i<allocpages;i++)//回收已分配的页
  {
     p=le2page(le,page_link);
     p->flags=0;
     p->property=1;
     SetPageProperty(p);
     le=list_next(le);
  }
  
  //实际的双链表信息复原后，还要对“二叉树”里面的节点信息进行更新
  node_size = 1;
  index = offset + self->size - 1;   //从原始的分配节点的最底节点开始改变longest
  self[index].longest = node_size;   //这里应该是node_size，也就是从1那层开始改变
  while (index) {//向上合并，修改父节点的记录值
    index = PARENT(index);
    node_size *= 2;
    left_longest = self[LEFT_LEAF(index)].longest;
    right_longest = self[RIGHT_LEAF(index)].longest;
    
    if (left_longest + right_longest == node_size) 
      self[index].longest = node_size;
    else
      self[index].longest = MAX(left_longest, right_longest);
  }
  for(i=pos;i<nr_block-1;i++)//清除此次的分配记录，即从分配数组里面把后面的数据往前挪
  {
    rec[i]=rec[i+1];
  }
  nr_block--;//更新分配块数的值
}
```

- 返回全局的空闲物理页数

```
static size_t
buddy_nr_free_pages(void) {
    return nr_free;
}
```

- 检查这个pmm_manager是否正确

```
//以下是一个测试函数
static void
buddy_check(void) {
    struct Page  *A, *B;
    A = B  =NULL;

    assert((A = alloc_page()) != NULL);
    assert((B = alloc_page()) != NULL);

    assert( A != B);
    assert(page_ref(A) == 0 && page_ref(B) == 0);
  //free page就是free pages(p0,1)
    free_page(A);
    free_page(B);
    
    A=alloc_pages(500);     //alloc_pages返回的是开始分配的那一页的地址
    B=alloc_pages(500);
    cprintf("A %p\n",A);
    cprintf("B %p\n",B);
    free_pages(A,250);     //free_pages没有返回值
    free_pages(B,500);
    free_pages(A+250,250);
    
}
```

# Challenge2：任意大小的内存单元slub分配算法(需要编程)

<blockquote>slub算法，实现两层架构的高效内存单元分配，第一层是基于页大小的内存分配，第二层是在 第一层基础上实现基于任意大小的内存分配。可简化实现，能够体现其主体思想即可。

参考linux的slub分配算法/，在ucore中实现slub分配算法。要求有比较充分的测试用例说 明实现的正确性，需要有设计文档。</blockquote>

slub和buddy system功能互补，buddy system按页分配内存空间，而slub则提供小内存空间的分配管理功能。

实际上 Slub 分配算法是非常复杂的，需要考虑缓存对齐、NUMA 等非常多的问题。简化的 Slub 算法结合了 Slab 算法和 Slub 算法的部分特征，使用了一些有技巧的实现方法。具体的简化为：

- Slab 大小为一页，不允许创建大对象仓库
- 复用Page数据结构，将 Slab 元数据保存在 Page 结构体中


每种对象由cache进行统一管理：

```
struct kmem_cache_t {
    list_entry_t slabs_full;	// 全满Slab链表
    list_entry_t slabs_partial;	// 部分空闲Slab链表
    list_entry_t slabs_free;	// 全空闲Slab链表
    uint16_t objsize;		// 对象大小
    uint16_t num;			// 每个Slab保存的对象数目
    void (*ctor)(void*, struct kmem_cache_t *, size_t);	// 构造函数
    void (*dtor)(void*, struct kmem_cache_t *, size_t);	// 析构函数
    char name[CACHE_NAMELEN];	// 仓库名称
    list_entry_t cache_link;	// 仓库链表
};
```

由于限制 Slub 大小为一页，所以数据对象和每页对象数据不会超过，所以使用 16 位整数保存足够。然后所有的cache链接成一个链表，方便进行遍历。

Slab
在上面的Buddy System中，一个物理页被分配之后，Page 结构中除了 ref 之外的成员都没有其他用处了，可以把Slab的元数据保存在这些内存中：

```
struct slab_t {
    int ref;				// 页的引用次数（保留）
    struct kmem_cache_t *cachep;	// 仓库对象指针
    uint16_t inuse;			// 已经分配对象数目
    int16_t free;			// 下一个空闲对象偏移量
    list_entry_t slab_link;		// Slab链表
};
```

为了方便空闲区域的管理，Slab 对应的内存页分为两部分：保存空闲信息的 bufcnt 以及可用内存区域 buf。

![](https://i.loli.net/2021/05/07/FpDzyxQa4i9scEr.png)

对象数据不会超过2048，所以 bufctl 中每个条目为 16 位整数。bufctl 中每个“格子”都对应着一个对象内存区域，不难发现，bufctl 保存的是一个隐式链表，格子中保存的内容就是下一个空闲区域的偏移，-1 表示不存在更多空闲区，slab_t 中的 free 就是链表头部。

内置仓库
除了可以自行管理仓库之外，操作系统往往也提供了一些常见大小的仓库，本次实验实现中内置了 8 个仓库，仓库对象大小为：8B、16B、32B、64B、128B、256B、512B、1024B。

操作函数

```
void *kmem_cache_grow(struct kmem_cache_t *cachep);
```

申请一页内存，初始化空闲链表 bufctl，构造 buf 中的对象，更新 Slab 元数据，最后将新的 Slab 加入到仓库的空闲Slab表中。

```
void kmem_slab_destroy(struct kmem_cache_t *cachep, struct slab_t *slab);
```

析构 buf 中的对象后将内存页归还。

公共函数

```
void kmem_int();
```

初始化 kmem_cache_t 仓库：由于 kmem_cache_t 也是由 Slab 算法分配的，所以需要预先手动初始化一个kmem_cache_t 仓库；

初始化内置仓库：初始化 8 个固定大小的内置仓库。

```
kmem_cache_create(const char *name, size_t size, void (*ctor)(void*, struct kmem_cache_t *, size_t),void (*dtor)(void*, struct kmem_cache_t *, size_t));
```

从 kmem_cache_t 仓库中获得一个对象，初始化成员，最后将对象加入仓库链表。其中需要注意的就是计算 Slab 中对象的数目，由于空闲表每一项占用 2 字节，所以每个 Slab 的对象数目就是：4096 字节/(2字节+对象大小)。

```
void kmem_cache_destroy(struct kmem_cache_t *cachep);
```

释放仓库中所有的 Slab，释放 kmem_cache_t。

```
void *kmem_cache_alloc(struct kmem_cache_t *cachep);
```

先查找 slabs_partial，如果没找到空闲区域则查找 slabs_free，还是没找到就申请一个新的 slab。从 slab 分配一个对象后，如果 slab 变满，那么将 slab 加入 slabs_full。

```
void *kmem_cache_zalloc(struct kmem_cache_t *cachep);
```

使用 kmem_cache_alloc 分配一个对象之后将对象内存区域初始化为零。

```
void kmem_cache_free(struct kmem_cache_t *cachep, void *objp);
```

将对象从 Slab 中释放，也就是将对象空间加入空闲链表，更新 Slab 元信息。如果 Slab 变空，那么将 Slab 加入slabs_partial 链表。

```
size_t kmem_cache_size(struct kmem_cache_t *cachep);
```

获得仓库中对象的大小。

```
const char *kmem_cache_name(struct kmem_cache_t *cachep);
```

获得仓库的名称。

```
int kmem_cache_shrink(struct kmem_cache_t *cachep);
```

将仓库中 slabs_free 中所有 Slab 释放。

```
int kmem_cache_reap();
```

遍历仓库链表，对每一个仓库进行 kmem_cache_shrink 操作。

```
void *kmalloc(size_t size);
```

找到大小最合适的内置仓库，申请一个对象。

```
void kfree(const void *objp);
```

释放内置仓库对象。

```
size_t ksize(const void *objp);
```

获得仓库对象大小。

实现过程如下：
```

--------------------------------------------------------------------------------------------
/*
slub.h
**/
--------------------------------------------------------------------------------------------
/*code*/
#ifndef __KERN_MM_SLUB_H__
#define  __KERN_MM_SLUB_H__

#include <pmm.h>
#include <list.h>

#define CACHE_NAMELEN 16

struct kmem_cache_t {
    list_entry_t slabs_full;
    list_entry_t slabs_partial;
    list_entry_t slabs_free;
    uint16_t objsize;
    uint16_t num;
    void (*ctor)(void*, struct kmem_cache_t *, size_t);
    void (*dtor)(void*, struct kmem_cache_t *, size_t);
    char name[CACHE_NAMELEN];
    list_entry_t cache_link;
};

struct kmem_cache_t *
kmem_cache_create(const char *name, size_t size,
                       void (*ctor)(void*, struct kmem_cache_t *, size_t),
                       void (*dtor)(void*, struct kmem_cache_t *, size_t));
void kmem_cache_destroy(struct kmem_cache_t *cachep);
void *kmem_cache_alloc(struct kmem_cache_t *cachep);
void *kmem_cache_zalloc(struct kmem_cache_t *cachep);
void kmem_cache_free(struct kmem_cache_t *cachep, void *objp);
size_t kmem_cache_size(struct kmem_cache_t *cachep);
const char *kmem_cache_name(struct kmem_cache_t *cachep);
int kmem_cache_shrink(struct kmem_cache_t *cachep);
int kmem_cache_reap();
void *kmalloc(size_t size);
void kfree(void *objp);
size_t ksize(void *objp);

void kmem_int();

#endif /* ! __KERN_MM_SLUB_H__ */
--------------------------------------------------------------------------------------------
/*
slub.c
**/
--------------------------------------------------------------------------------------------
/*code*/
#include <slub.h>
#include <list.h>
#include <defs.h>
#include <string.h>
#include <stdio.h>

struct slab_t {
    int ref;                       
    struct kmem_cache_t *cachep;              
    uint16_t inuse;
    uint16_t free;
    list_entry_t slab_link;
};

// The number of sized cache : 16, 32, 64, 128, 256, 512, 1024, 2048
#define SIZED_CACHE_NUM     8
#define SIZED_CACHE_MIN     16
#define SIZED_CACHE_MAX     2048

#define le2slab(le,link)    ((struct slab_t*)le2page((struct Page*)le,link))
#define slab2kva(slab)      (page2kva((struct Page*)slab))

static list_entry_t cache_chain;
static struct kmem_cache_t cache_cache;
static struct kmem_cache_t *sized_caches[SIZED_CACHE_NUM];
static char *cache_cache_name = "cache";
static char *sized_cache_name = "sized";

// kmem_cache_grow - add a free slab
static void * kmem_cache_grow(struct kmem_cache_t *cachep) {
    struct Page *page = alloc_page();
    void *kva = page2kva(page);
    // Init slub meta data
    struct slab_t *slab = (struct slab_t *) page;
    slab->cachep = cachep;
    slab->inuse = slab->free = 0;
    list_add(&(cachep->slabs_free), &(slab->slab_link));
    // Init bufctl
    int16_t *bufctl = kva;
    for (int i = 1; i < cachep->num; i++)
        bufctl[i-1] = i;
    bufctl[cachep->num-1] = -1;
    // Init cache 
    void *buf = bufctl + cachep->num;
    if (cachep->ctor) 
        for (void *p = buf; p < buf + cachep->objsize * cachep->num; p += cachep->objsize)
            cachep->ctor(p, cachep, cachep->objsize);
    return slab;
}

// kmem_slab_destroy - destroy a slab
static void kmem_slab_destroy(struct kmem_cache_t *cachep, struct slab_t *slab) {
    // Destruct cache
    struct Page *page = (struct Page *) slab;
    int16_t *bufctl = page2kva(page);
    void *buf = bufctl + cachep->num;
    if (cachep->dtor)
        for (void *p = buf; p < buf + cachep->objsize * cachep->num; p += cachep->objsize)
            cachep->dtor(p, cachep, cachep->objsize);
    // Return slub page 
    page->property = page->flags = 0;
    list_del(&(page->page_link));
    free_page(page);
}

static int kmem_sized_index(size_t size) {
    // Round up 
    size_t rsize = ROUNDUP(size, 2);
    if (rsize < SIZED_CACHE_MIN)
        rsize = SIZED_CACHE_MIN;
    // Find index
    int index = 0;
    for (int t = rsize / 32; t; t /= 2)
        index ++;
    return index;
}

// ! Test code
#define TEST_OBJECT_LENTH 2046
#define TEST_OBJECT_CTVAL 0x22
#define TEST_OBJECT_DTVAL 0x11

static const char *test_object_name = "test";

struct test_object {
    char test_member[TEST_OBJECT_LENTH];
};

static void test_ctor(void* objp, struct kmem_cache_t * cachep, size_t size) {
    char *p = objp;
    for (int i = 0; i < size; i++)
        p[i] = TEST_OBJECT_CTVAL;
}

static void test_dtor(void* objp, struct kmem_cache_t * cachep, size_t size) {
    char *p = objp;
    for (int i = 0; i < size; i++)
        p[i] = TEST_OBJECT_DTVAL;
}

static size_t list_length(list_entry_t *listelm) {
    size_t len = 0;
    list_entry_t *le = listelm;
    while ((le = list_next(le)) != listelm)
        len ++;
    return len;
}

static void check_kmem() {

    assert(sizeof(struct Page) == sizeof(struct slab_t));

    size_t fp = nr_free_pages();

    // Create a cache 
    struct kmem_cache_t *cp0 = kmem_cache_create(test_object_name, sizeof(struct test_object), test_ctor, test_dtor);
    assert(cp0 != NULL);
    assert(kmem_cache_size(cp0) == sizeof(struct test_object));
    assert(strcmp(kmem_cache_name(cp0), test_object_name) == 0);
    // Allocate six objects
    struct test_object *p0, *p1, *p2, *p3, *p4, *p5;
    char *p;
    assert((p0 = kmem_cache_alloc(cp0)) != NULL);
    assert((p1 = kmem_cache_alloc(cp0)) != NULL);
    assert((p2 = kmem_cache_alloc(cp0)) != NULL);
    assert((p3 = kmem_cache_alloc(cp0)) != NULL);
    assert((p4 = kmem_cache_alloc(cp0)) != NULL);
    p = (char *) p4;
    for (int i = 0; i < sizeof(struct test_object); i++)
        assert(p[i] == TEST_OBJECT_CTVAL);
    assert((p5 = kmem_cache_zalloc(cp0)) != NULL);
    p = (char *) p5;
    for (int i = 0; i < sizeof(struct test_object); i++)
        assert(p[i] == 0);
    assert(nr_free_pages()+3 == fp);
    assert(list_empty(&(cp0->slabs_free)));
    assert(list_empty(&(cp0->slabs_partial)));
    assert(list_length(&(cp0->slabs_full)) == 3);
    // Free three objects 
    kmem_cache_free(cp0, p3);
    kmem_cache_free(cp0, p4);
    kmem_cache_free(cp0, p5);
    assert(list_length(&(cp0->slabs_free)) == 1);
    assert(list_length(&(cp0->slabs_partial)) == 1);
    assert(list_length(&(cp0->slabs_full)) == 1);
    // Shrink cache 
    assert(kmem_cache_shrink(cp0) == 1);
    assert(nr_free_pages()+2 == fp);
    assert(list_empty(&(cp0->slabs_free)));
    p = (char *) p4;
    for (int i = 0; i < sizeof(struct test_object); i++)
        assert(p[i] == TEST_OBJECT_DTVAL);
    // Reap cache 
    kmem_cache_free(cp0, p0);
    kmem_cache_free(cp0, p1);
    kmem_cache_free(cp0, p2);
    assert(kmem_cache_reap() == 2);
    assert(nr_free_pages() == fp);
    // Destory a cache 
    kmem_cache_destroy(cp0);

    // Sized alloc 
    assert((p0 = kmalloc(2048)) != NULL);
    assert(nr_free_pages()+1 == fp);
    kfree(p0);
    assert(kmem_cache_reap() == 1);
    assert(nr_free_pages() == fp);

    cprintf("check_kmem() succeeded!\n");

}
// ! End of test code

// kmem_cache_create - create a kmem_cache
struct kmem_cache_t * kmem_cache_create(const char *name, size_t size,
                       void (*ctor)(void*, struct kmem_cache_t *, size_t),
                       void (*dtor)(void*, struct kmem_cache_t *, size_t)) {
    assert(size <= (PGSIZE - 2));
    struct kmem_cache_t *cachep = kmem_cache_alloc(&(cache_cache));
    if (cachep != NULL) {
        cachep->objsize = size;
        cachep->num = PGSIZE / (sizeof(int16_t) + size);
        cachep->ctor = ctor;
        cachep->dtor = dtor;
        memcpy(cachep->name, name, CACHE_NAMELEN);
        list_init(&(cachep->slabs_full));
        list_init(&(cachep->slabs_partial));
        list_init(&(cachep->slabs_free));
        list_add(&(cache_chain), &(cachep->cache_link));
    }
    return cachep;
}

// kmem_cache_destroy - destroy a kmem_cache
void kmem_cache_destroy(struct kmem_cache_t *cachep) {
    list_entry_t *head, *le;
    // Destory full slabs
    head = &(cachep->slabs_full);
    le = list_next(head);
    while (le != head) {
        list_entry_t *temp = le;
        le = list_next(le);
        kmem_slab_destroy(cachep, le2slab(temp, page_link));
    }
    // Destory partial slabs 
    head = &(cachep->slabs_partial);
    le = list_next(head);
    while (le != head) {
        list_entry_t *temp = le;
        le = list_next(le);
        kmem_slab_destroy(cachep, le2slab(temp, page_link));
    }
    // Destory free slabs 
    head = &(cachep->slabs_free);
    le = list_next(head);
    while (le != head) {
        list_entry_t *temp = le;
        le = list_next(le);
        kmem_slab_destroy(cachep, le2slab(temp, page_link));
    }
    // Free kmem_cache 
    kmem_cache_free(&(cache_cache), cachep);
}   

// kmem_cache_alloc - allocate an object
void * kmem_cache_alloc(struct kmem_cache_t *cachep) {
    list_entry_t *le = NULL;
    // Find in partial list 
    if (!list_empty(&(cachep->slabs_partial)))
        le = list_next(&(cachep->slabs_partial));
    // Find in empty list 
    else {
        if (list_empty(&(cachep->slabs_free)) && kmem_cache_grow(cachep) == NULL)
            return NULL;
        le = list_next(&(cachep->slabs_free));
    }
    // Alloc 
    list_del(le);
    struct slab_t *slab = le2slab(le, page_link);
    void *kva = slab2kva(slab);
    int16_t *bufctl = kva;
    void *buf = bufctl + cachep->num;
    void *objp = buf + slab->free * cachep->objsize;
    // Update slab
    slab->inuse ++;
    slab->free = bufctl[slab->free];
    if (slab->inuse == cachep->num)
        list_add(&(cachep->slabs_full), le);
    else 
        list_add(&(cachep->slabs_partial), le);
    return objp;
}

// kmem_cache_zalloc - allocate an object and fill it with zero
void * kmem_cache_zalloc(struct kmem_cache_t *cachep) {
    void *objp = kmem_cache_alloc(cachep);
    memset(objp, 0, cachep->objsize);
    return objp;
}

// kmem_cache_free - free an object
void kmem_cache_free(struct kmem_cache_t *cachep, void *objp) {
    // Get slab of object 
    void *base = page2kva(pages);
    void *kva = ROUNDDOWN(objp, PGSIZE);
    struct slab_t *slab = (struct slab_t *) &pages[(kva-base)/PGSIZE];
    // Get offset in slab
    int16_t *bufctl = kva;
    void *buf = bufctl + cachep->num;
    int offset = (objp - buf) / cachep->objsize;
    // Update slab 
    list_del(&(slab->slab_link));
    bufctl[offset] = slab->free;
    slab->inuse --;
    slab->free = offset;
    if (slab->inuse == 0)
        list_add(&(cachep->slabs_free), &(slab->slab_link));
    else 
        list_add(&(cachep->slabs_partial), &(slab->slab_link));
}

// kmem_cache_size - get object size
size_t kmem_cache_size(struct kmem_cache_t *cachep) {
    return cachep->objsize;
}

// kmem_cache_name - get cache name
const char * kmem_cache_name(struct kmem_cache_t *cachep) {
    return cachep->name;
}

// kmem_cache_shrink - destroy all slabs in free list 
int kmem_cache_shrink(struct kmem_cache_t *cachep) {
    int count = 0;
    list_entry_t *le = list_next(&(cachep->slabs_free));
    while (le != &(cachep->slabs_free)) {
        list_entry_t *temp = le;
        le = list_next(le);
        kmem_slab_destroy(cachep, le2slab(temp, page_link));
        count ++;
    }
    return count;
}

// kmem_cache_reap - reap all free slabs 
int kmem_cache_reap() {
    int count = 0;
    list_entry_t *le = &(cache_chain);
    while ((le = list_next(le)) != &(cache_chain))
        count += kmem_cache_shrink(to_struct(le, struct kmem_cache_t, cache_link));
    return count;
}

void * kmalloc(size_t size) {
    assert(size <= SIZED_CACHE_MAX);
    return kmem_cache_alloc(sized_caches[kmem_sized_index(size)]);
}

void kfree(void *objp) {
    void *base = slab2kva(pages);
    void *kva = ROUNDDOWN(objp, PGSIZE);
    struct slab_t *slab = (struct slab_t *) &pages[(kva-base)/PGSIZE];
    kmem_cache_free(slab->cachep, objp);
}

void kmem_int() {

    // Init cache for kmem_cache
    cache_cache.objsize = sizeof(struct kmem_cache_t);
    cache_cache.num = PGSIZE / (sizeof(int16_t) + sizeof(struct kmem_cache_t));
    cache_cache.ctor = NULL;
    cache_cache.dtor = NULL;
    memcpy(cache_cache.name, cache_cache_name, CACHE_NAMELEN);
    list_init(&(cache_cache.slabs_full));
    list_init(&(cache_cache.slabs_partial));
    list_init(&(cache_cache.slabs_free));
    list_init(&(cache_chain));
    list_add(&(cache_chain), &(cache_cache.cache_link));

    // Init sized cache 
    for (int i = 0, size = 16; i < SIZED_CACHE_NUM; i++, size *= 2)
        sized_caches[i] = kmem_cache_create(sized_cache_name, size, NULL, NULL); 

    check_kmem();
}
```
最终challenge结果：
![](https://i.loli.net/2021/05/09/7JipHlPNuO9Rzdr.png)