
# 技术备忘录

这里将记录并分享一些我在实际编程或项目中遇到的一些问题，和我解决这些问题的思路和想法，当然也会分享一些有关linux内核，和linux实际常用的小skill。

------------

## Linux缺页中断处理

Linux缺页中断处理

CPU将一个线性地址映射为物理地址的过程中,如果该地址的映射已经建立,但是发现相应的页面表项或目录项中的P(Present)标志为0,则表明相应的物理页面不在内存.此时,CPU将错误的请求地址放在寄存器CR2中,并发生缺页中断.
穿过了中断门后,控制直接到达程序入口page_fault(),该入 口代码将页面失败处理例程do_page_fault()的地址压栈,并调用通用的错误处理例程保存现场并使控制转移到do_page_fault(). 

至此,完成了软硬件的交接.

对于硬件来说,CPU发生缺页中断即认为指令访问了无效的地址空间.
但是,从操作系统的角度,其原因可能是多样的.而且,内核的中断处理程序也可能进入do_page_fault(),在这里我们只考虑页面失败发生在当前进程的情况.

在Linux中,控制进入 do_page_fault()的原因可能有:
1,真实的地址越界,即进程访问了不属于自己的地址空间;
2,页面已经映射,但未载入内存或者已被换出;
3,进程页面未被映射,即相应的页表不存在.

通过判断currentTask->mm是否为空,就可确定是进程的内存是否还未映射,如果还未映射,则控制将转到no_context.
否则,为了区分越界还是缺页,do_page_fault()首先从CR2中读出出错的地址address,然后调用find_vma(mm, address),在进程的虚拟地址空间中找出结束地址大于address的第一个区间.如果找不到的话,则说明异常是由地址越界引起的,转到bad_area执行相关错误处理.

而下面我们将重点讨论页面未映射或被换出这两种情况.

确定并非地址越界后,控制转向标号good_area. 在这里,代码首先对页面进行例行权限检查,比如当前的操作是否违反该页面的Read,Write,Exec权限等等.

如果通过了检查,则进入虚存管理例程handle_mm_fault().否则,将与地址越界一样,转到bad_area继续处理.

handle_mm_fault()用于实现页面分配与交换.它分为两个步骤:
首先,如果页表不存在或被交换出,则要首先分配页面给页表;然后,才真正实施页面的分配,并在页表上做记录.

handle_mm_fault()首先求出address所属的页目录项指针pgd,然后调用pte_alloc(pgd),pte_alloc()会首先判断该目录项对应的页表是否存在,如果不存在的话就分配一个新的页表并返回.
pte_alloc()分配页表时,会首先调用get_pte_fast(),试图从页面缓冲池中获取页表空间,而如果缓冲池空,则调用get_pte_kernel_slow()从磁盘上获取用于存放页表的页面.
然后,pte_alloc()会调用set_pmd()初始化这个页表,即将该页表的起始地址,标志位等写入页目录项pgd.
到此为止,页表pte的空间已分配并初始化,但并不包含任何内容.接着,handle_mm_fault()将调用handle_pte_fault()来具体分配页面并填充页表pte.

handle_pte_fault() 是实现虚存管理的核心部分,它实现了页面的分配和交换.
它首先调用pte_present()检查表项中的P标志位,当然,在我们的讨论中它总是返回false.
接着,又调用pte_none()检查表项是否为空,即全为0.如果为空就说明映射尚未建立,此时调用do_no_page()来建立内存页面与交换文件的映射;反之,如果表项非空,则说明页面已经映射,只要调用do_swap_page()将其换入内存即可.

do_no_page()除了能够分配物理页面外,还用于实现内存映射文件及共享页面等.在我们讨论的缺页中断中,它只会调用do_anonymous_page()分配一个新的物理页面,并设置相应的页表项.此外,该过程还会进行一些权限检查,在此略过.do_anonymouse_page()是通过调用alloc_pages()完成实际内存分配的.如果分配成功,则整个缺页中断处理过程完成,所有过程依次退栈返回.

当控制返回到用户空间后,因为缺页而夭折的那条指令会被重新执行.从用户程序的角度来看,这一切就像没有发生过一样,缺页中断处理对用户是透明的.

上面讨论的是页面未映射的情况.如果已经完成了页面映射,而缺页是由于页面被换出而引起的,则应调用do_swap_page()将从交换设备上换入页面.

为了研究Linux的页面置换方法,必须先弄清相关的数据结构.

众所周知,对于i386系列处理器,当页面在内存中时,页表项是一个32位的pte_t结构,它描述了物理页面起始地址,页面权限标志等信息,这些信息将被 CPU用于地址转换.
但是,这一切的前提是,该表项的P标志位为1.否则,这个32位的页表项将被CPU视为无意义的.
但是对于操作系统而言,当它换出页面之时,应该在设置P标志为0的同时,将页面对应的磁盘页信息放入页表项,以供调回页面时获取.即同样一个32位的页表项,在P标志位不同时,实际上对应着不同的存储内容.

就Linux而言,如果页表项的P标志位0,则该页表项存放的实际上是一个swap_entry_t结构,它确定了该页面在磁盘上唯一位置,包括用于交换的文件或设备的序号,以及页面在此文件或设备中的相对位置.

do_swap_page()会首先调用lookup_swap_cache(),判断相应的内存页面是否还留在swapper_space的换入/换出队列中尚未最后释放.
如果确实已经释放, 则通过read_swap_cache()分配一个内存页面,并从磁盘中读取先前交换出的页面内容到分配的页中,当然,要读取的磁盘块是从"页表项"中的swap_entry_t结构获得的.
同时,出于性能考虑,还会调用swapin_readahead()将一些临近的页面顺带一起读入上面提到的 swapper_space中,与此相关还涉及到一些复杂的异步读取操作,在此略过不提.
最后,do_swap_page()还要设置页表项的一些标志信息(例如P,D等),访问权限等.
最后,如同do_no_page()执行完毕后一样,返回到用户空间,重新执行因缺页而夭折的指令.

上面已经完 整描述了Linux缺页中断处理的大致流程,但并没有详细描述页面交换时的换入/换出算法.
事实上,其中的 alloc_page,read_swap_cache等调用都可能导致其他页面被换出.
至于Linux是如何淘汰页面的,是一个比较复杂的话题,但大体说来它所采用的淘汰策略是LRU.

------------

## 关于异构内存细粒度内存分配实现方法的思想


![](https://github.com/Gumi-presentation-by-Dzh/Fxxk-my-code/raw/master/image/1.jpg)


### 内核层

[相关资料](http://blog.csdn.net/vanbreaker/article/details/7855007)

在VMA中将新的NVM_VMA标记进去
——在linux/mm_type中可以找到VMA的定义vm_area_struct
————结构体中有一部分是unsigned long vm_flags;        /* Flags, see mm.h. */   这个地方可以理解成标记

可以在mm.h中看到一些对于vm_flags的定义，也就是说要在这里加入NVM_VMA标记

```markdown
/* Is the vma a continuation of the stack vma above it? */
static inline int vma_growsdown(struct vm_area_struct *vma, unsigned long addr)
{
    return vma && (vma->vm_end == addr) && (vma->vm_flags & VM_GROWSDOWN);
}
```

可以看到之后的操作会用vm_flags与定义事件VM_……与，等到逻辑值。
```markdown
#define VM_HUGETLB    0x00400000    /* Huge TLB Page VM */
#define VM_NONLINEAR    0x00800000    /* Is non-linear (remap_file_pages) */
类似定义出VM_NVM
```

```markdown
实际内核定义在  linux/mm.h中
其值为#define VM_NVM          0x08000000      /*marked this vma to use NVM malloc*/
```

![](https://github.com/Gumi-presentation-by-Dzh/Fxxk-my-code/raw/master/image/2.jpg)
![](https://github.com/Gumi-presentation-by-Dzh/Fxxk-my-code/raw/master/image/3.jpg)


参照mmap，添加nvm_mmap分支    （pmfs/mm/mmap.c  do_mmap_pgoff()  do_mmap()）
nvm_mmap为vm_flags添加NVM_VMA标记（也就是定义在mm.h中的）
```markdown
vm_flags = calc_vm_prot_bits(prot) | calc_vm_flag_bits(flags) | mm->def_flags |
VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;      --设置vm_flags，根据传入的port和flags以及mm本身自有的旗标来设置。
```

### 函数调用流程

用户态：
malloc调用mmap————>addr = mmap(NULL, 4096, PROT_READ|PORT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
//mmap(void* start, size_t length, int prot, int flags, int fd, off_t offset);
对于malloc映射匿名内存来说，必须是以页为单位的，比如上面的4096。用户进程向内核空间分配内存都是直接向伙伴系统要的。在此基础上glibc将内存细化为可以按照字节分配的方式。
mmap调用sys_mmap2（简单转调用）

内核态：
inline函数do_mmap()，是供内核自己用的，它也是将已打开文件映射到当前进程空间。
在PMFS中的实现部分为 if(file)之后的部分对flag进行设置。

```markdown
    if (flags & MAP_NORESERVE) {              //可以直接利用这种模板来设计接口，但是要补货上层代码传入时候的时候flags的定义（即MAP_NORESERVE）的定义

        /* hugetlb applies strict overcommit unless MAP_NORESERVE */

        if (file && is_file_hugepages(file))

            vm_flags |= VM_NORESERVE;

    }
```

参数倒入可以查找到导入的头文件，可以找到

```markdown
#include <linux/mman.h>
查找到/include/linux/mman.h
又调用了#include <uapi/linux/mman.h>
查找到/include/uapi/linux/mman.h
之后可以在/include/uapi/asm-generic/mman.h 找到一些定义        （可能需要在这里加入flag参数的定义，可以认定在此添加NVM标记）

include/linux/syscalls.h有大部分syscall的定义
include/asm-generic/syscalls.h  可以找到sys_mmap2函数和sys_mmap.  调用（do_mmap2跳转sys_mmap_pgoff） ----->调用了sys_mmap_pgoff之后没函数定义了（最终归属）该被定义在include/linux/syscalls.h声明，推断调用内核的do_mmap_pgoff
```

由此可以推定：

从stdlib来的malloc调用了mmap函数（网上说的是系统调用）

直接会走对应的系统调用sys_mmap2（或sys_mmap）（系统调用）  途径do_mmap2直接调用内核的do_mmap_pgoff（网上的说法）（但内核中找不到网上提及的do_map内敛函数-PMFS内核linux3.11.0）

但从PMFS内核分析可以看出，这个sys_mmap一般不会通过do_mmap2，而是调用syscall里面的sys_mmap_pgoff.此时极大可能通过此函数直接调用内核do_mmap_pgoff

也就是说最终在内核态中都会使用do_mmap_pgoff
则NVM内存分配应该在do_mmap_pgoff关联的函数中实现（例如pagefault（中科院思想））。

何时，怎样对VM_flag设置成NVMtpye则需要考虑

可以看出  mmap系列的传参数  很关键       （PMFS内设了MAP_ANONYMOUS | MAP_HUGETLB这两个特殊VM_FLAG参数，有极高的参考价值）
在pmfs/tools/testing/selftests/vm/hugetlbfstest.c  可以看到这个设定。

```markdown
代码实际实现：
在/mm/mmap中实现对VM_NVM的标记
其中需要通过系统调用的mmap参数中的flag，而系统调用的mmap flag需要在/中设置
    vm_flags = calc_vm_prot_bits(prot) | calc_vm_flag_bits(flags) | mm->def_flags |
               VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;      --设置vm_flags，根据传入的port和flags以及mm本身自有的旗标来设置。

之后if (flags & MAP_NVM)
            vm_flags |= VM_NVM;

pmfs/include/uapi/asm-generic/mman-common.h添加MAP_NVM

#define MAP_NVM         0x80000         /*creat a nvm page mapping*/

#ifndef MAP_NVM
#define MAP_NVM         0x80000
#endif
```

Pagefault时，对标记位NVM_VMA的VMA，为其在node1节点上分配内存
——vm_operations_struct  在linux/mm.h/int (*fault)(struct vm_area_struct *vma, struct vm_fault *vmf);/vma->vm_ops->fault（非重点）

mm/fault.c
do_page_fault()函数.

page fault基本流程：
从cr2中获取发生异常的地址
——缺页地址位于内核态
————位于vmalloc区？->从主内核页表同步数据到进程页表
————非vmalloc区 ->不应该产生page fault->oops
——缺页地址位于用户态
————缺页上下文发生在内核态
——————exception table中有相应的处理项？ ->进行修正
——————没有 ->oops
————查找vma
——————找到？-> 是否expand stack？->堆栈扩展
—————————— 不是->正常的缺页处理：handle_mm_fault                    
——————没找到->bad_area  

仿佛是通过  fault = handle_mm_fault(mm, vma, address, flags); 来实现的分配。  //密切关注 handle_pte_fault，并且要了明确了解handle_mm_fault的实现原理。

大致流程中分为：
地址为内核空间：
1，当地址为内核地址空间并且在内核中访问时，如果是非连续内存地址，将init_mm中对应的项复制到本进程对应的页表项做修正；
2，地址为内核空间时，检查页表的访问权限；
3，如果1,2没搞定，跳到非法访问处理（在后面详细分析这个）；
地址为用户空间：
4，如果使用了保留位，打印信息，杀死当前进程；
5，如果在中断上下文中火临界区中时，直接跳到非法访问；
6，如果出错在内核空间中，查看异常表，进行相应的处理；
7，查找地址对应的vma，如果找不到，直接跳到非法访问处，如果找到正常，跳到good_area；
8，如果vma->start_address>address，可能是栈太小，对齐进行扩展；
9，good_area处，再次检查权限；
10，权限正确后分配新页框，页表等；

```markdown
do_page_fault()调用handle_mm_fault()调用handle_pte_fault()（do_no_page()被省略了（似乎））调用do_anonymous_page(),

arch/x86/mm/fault.c——————>do_page_fault()

pmfs/mm/memory.c——————>handle_mm_fault()
出现了do_pmd_numa_page()也在这个文档里面

pmfs/mm/memory.c——————>handle_pte_fault()
出现了do_numa_page()也在这个文档里面

pmfs/mm/memory.c中并没有定义do_no_page()，项目中也没有定义这个函数

pmfs/mm/memory.c——————>do_anonymous_page()

触发读写异常然后新分配一页,将相关属性写入到页面表项中

节点分配页面貌似调用的是alloc_pages     pmfs/include/linux/gfp.h
```

------------

## Glibc安装

```markdown
mkdir glibc-build-2.15    //不能再src里面装

cd glibc-build-2.15

../glibc-2.19/configure  --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin

make >buildtest

make install

```

[相关资料](http://blog.csdn.net/officercat/article/details/39520227)

------------

## 内核安装编译，启动项改变，和内核删除

### 内核删除

```markdown
   uname -r       该命令会告诉你当前使用的内核版本
```

接下来，如果你是自己动手编译的内核的话，请删除以下文件和文件夹

1. 删除掉/lib/modules/目录下过时的内核库文件
2. 删除掉/usr/src/kernels/目录下过时的内核源代码
3. 删除掉/boot目录下启动的核心档案以及内核映像
4. 更改/boot/grub/menu.lst，删除掉不用的启动列表

### 内核编译安装

```markdown
make mrproper #清理上次编译的现场 

cp /boot/config-($uname -r) .config #将当前系统的配置文件拷贝到当前目录

sh -c 'yes "" | make oldconfig' #使用旧内核配置，并自动接受每个新增选项的默认设置

sudo make -j20 bzImage #生成内核文件
sudo make -j20 modules #编译模块
sudo make -j20 modules_install #编译安装模块

sudo make install #内核安装
```

### 修改启动项

```markdown
sudo vim /etc/default/grub

memmap=16G\\\$4G

sudo grub2-mkconfig -o /boot/grub2/grub.cfg //生成grub2的配置文件
```

------------

## git常用技巧

git init （初始化本地仓库）

git remote add github git@******

将本地仓库github（add后面的名字）和远程仓库链接

git pull github master将远程仓库分支master与本地分支github合并（如果本地为空等于复制项目到本地）如果在diff或者show 可以起到更新本地代码的效果

git status 查看本地仓库当前分支是否由commit

git commit -m “”   向本地仓库提交commit

git log  查看本地git日志

git diff  可以查看与远端仓库的不同

git show可以查看push之前的状况

git push github master  将本地github分支推送到远端master分支

git add .    添加添加 .意味所有

当你安装Git后首先要做的事情是设置你的用户名称和e-mail地址。这是非常重要的，因为每次Git提交都会使用该信息。它被永远的嵌入到了你的提交中：

git config --global user.name "John Doe"
git config --global user.email johndoe@example.com 

[相关资料1](http://www.yiibai.com/git/git_managing_branches.html)
[相关资料2](http://blog.csdn.net/hhhccckkk/article/details/50737077)

------------

## linux常用技巧

### core-debug和gdb

```markdown
gcc -g -rdynamic x.c
gdb ./a.out
```

### 文件夹内容复制

```markdown
cp -rf glibc-2.19/* NVM_malloc/ #*代表所有子目录文件
```

### nm命令查看目标文件符号清单

```markdown
nm -A libso.a
```

### 后台进程挂起

```markdown
nohup command > myout.file & #命令行  输出到  myout.file 
ps -aux |grep "git"  #查看挂起进程
```

### 添加linux用户

```markdown
useradd -d /home/ZHduan -m ZHduan
passwd ZHduan

chmod 777 /etc/sudoers
vim /etc/sudoers
chmod 440 /etc/sudoers

ls -al 检查一下 当然可以直接grep sudoers
```

------------

## Support and Contact
ZHduan [个人主页](https://gumi-presentation-by-dzh.github.io/Myresume/index.html)
Email 122316931@qq.com


