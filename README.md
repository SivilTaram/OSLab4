# OSLab4
BUAA OSLab4
##实验概况##
***
  在开始实验之前，先对实验整体有个大概的了解，这样能让我们更好地进行实验。
  我们本次实验需要补充的内容包括一整套以sys开头的系统调用函数，其中包括了进程间通信需要的一些系统调用如sys_ipc_can_recv等，以及补充完成fork.c函数，当然也不能少填写`syscall_wrap.S`.
##系统调用##
***
  关于系统调用，我们主要是以以下流程来进行的：  
  + 用户调用`syscall`特权指令触发异常
  + 异常触发，`pc`值自动被硬件置为`0x80000080`，转向异常分发代码
  + trap_init识别是系统调用(8号异常)，为其分配处理函数`handle_sys`
  + 将用户态下的参数拷贝到内核中，根据第1个参数为索引寻找系统调用表`syscalltable`
  + 根据系统调用号在`syscalltable`中找到相应的函数后，转向对应函数处理
  + 处理完成后，回到用户态，系统调用完成。

```C
LEAF(msyscall)

/////////////////////////////////////////
//insert your code here
//put the paramemters to the right pos
//and then call the "syscall" instruction
/////////////////////////////////////////
//fmars:
//j fmars
  move v0,a0
  sw a0,0(sp)
  sw a1,4(sp)
  sw a2,8(sp)
  sw a3,12(sp)
  syscall
  jr ra


END(msyscall)

//move a0->v0
//move a1-3 to stack

```
上面是我们这次要填写的`./user/syscall_wrap.S`函数，结合注释与我们上面的讲解，应该不难理解。实际上就是如下流程：1、设置`syscall`的参数；2、执行`syscall`;3、完成系统调用，返回<br>
而在mips下有如下约定：<br>
```
v0         用于置系统调用号
a0~a3      置前四个参数，后面的参数用栈传
syscall    系统调用触发指令
```
在这一阶段你可能存在的困惑是，在每个函数中出现的`int sysno`的参数究竟有什么用?实际上笔者认为这个参数并没有什么用处,不过笔者最后认为,`sysno`其实就是`a0`的值,其实也就是我们的系统调用号。我们可以在`./user/syscall_lib.c`里面看到其引用:
```C
int
syscall_set_pgfault_handler(u_int envid, u_int func, u_int xstacktop)
{
        return msyscall(SYS_set_pgfault_handler,envid,func,xstacktop,0,0);
}
```
这就又牵涉到一个问题,`syscall_x`和`sys_x`函数有什么关联和区别呢?  
      对于这个问题，实际上，在后面填写`fork.c`的时候,我们可以发现我们使用的函数全部都是`syscall_x`类的函数，而不使用`sys_x`类的函数。实际上根据我们上面的流程，调用关系是这样的：
```
  * fork.c中调用syscall_x类的函数
  * syscall_x调用mysyscall汇编函数
  * mysyscall汇编函数中调用了syscall特权指令。
  * syscall指令根据调用号选择sys_x类的函数
```   
填完系统调用后，就可以开始填写跟系统调用有关的`syscalltable`中的系统调用子函数了。在Lab4中这些子函数注释严重匮乏，所以我参考了MIT的JOS的注释来进行理解和填写。
###sys_set_pgfault_handler###
第一个要补全的函数是这个，先来看看MIT原生注释是怎样的：
```C
// Set the page fault upcall for 'envid' by modifying the corresponding struct
// Env's 'env_pgfault_upcall' field.  When 'envid' causes a page fault, the
// kernel will push a fault record onto the exception stack, then branch to
// 'func'.
//为envid所对应的进程控制块设立对应的缺页处理函数，通过修改进程控制块通信结构中的 'env_pgfault_upcall'区域  
//(在我们的实验中是 env_pgfault_handler )。当 envid 进程造成页缺失时，内核将会把页缺失记录入异常栈  
//(exceptionstack),然后转向处理函数 'func'。
// Returns 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
```
实际上面这段注释已经说明了这个函数的作用，实际上函数应该还是比较好填写的。结合注释和指导书中的内容，应该能轻松补全。

###int sys_mem_alloc(int sysno,u_int envid, u_int va, u_int perm)###
同样，上手先参考一下MIT-JOS的原生注释：
```C
// Allocate a page of memory and map it at 'va' with permission
// 'perm' in the address space of 'envid'.
// The page's contents are set to 0.
// If a page is already mapped at 'va', that page is unmapped as a
// side effect.
// 
// 分配一页内存在'envid'进程对应的地址空间中，让'va'以'perm'的权限位映射它。新分配的那页内容要清零。如果已有一个va映射
//到了该页，那么要解映射。
//
// perm -- PTE_U | PTE_P must be set, PTE_AVAIL | PTE_W may or may not be set,
//         but no other bits may be set.  See PTE_SYSCALL in inc/mmu.h.
//权限位PTE_U | PTE_P 必须被给予，PTE_AVAIL | PTE_W 给不给都可以，
//但是不会有其他权限位
// Return 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
//	-E_INVAL if va >= UTOP, or va is not page-aligned.
//	-E_INVAL if perm is inappropriate (see above).
//	-E_NO_MEM if there's no memory to allocate the new page,
//		or to allocate any necessary page tables.
```
关于所有权限位的解释与说明，我们可以参考MIT-JOS的注释，可以发现：
```C
#define PTE_P		0x001	// Present
//PTE_P和我们的PTE_V的作用一致，表明一个页表项(或者页目录项)是有效的。

#define PTE_W		0x002	// Writeable
//PTE_W和我们的PTE_R的作用一致，表明该页表项对应的页是用户可写的。

#define PTE_U		0x004	// User
//PTE_U是我认为一个我们实验需要但是并没有被定义的权限位。

#define PTE_D		0x040	// Dirty
//PTE_D为什么没有定义，难道不需要写回磁盘？因为我们不会对文件进行修改？

#define PTE_COW		0x800	// Avail for system programmer's use

// The PTE_AVAIL bits aren't used by the kernel or interpreted by the
// hardware, so user processes are allowed to set them arbitrarily.
#define PTE_AVAIL	0xE00	// Available for software use

// Flags in PTE_SYSCALL may be used in system calls.  (Others may not.)
#define PTE_SYSCALL	(PTE_AVAIL | PTE_P | PTE_W | PTE_U)
```

下面是我们实验中的权限位的设置
```C
#define PTE_V           0x0200  // Valid bit
#define PTE_R           0x0400  // Dirty bit ,'0' means only read ,otherwise make interrupt
#define PTE_COW         0x0001  // Copy On Write
#define PTE_LIBRARY             0x0004  // share memmory
```
很多我们实验中没有见到却定义了的的`PTE_D`,`PTE_UC`,`PTE_G`都没有出现，以上四个权限位是我们贯穿所有实验的最重要的几个权限位。


```C

```
