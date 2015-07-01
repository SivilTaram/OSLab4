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
//为envid所对应的进程控制块设立对应的缺页处理函数，通过修改进程控制块通信结构中的 'env_pgfault_upcall'区域  (在我们的实验中是 env_pgfault_handler )。当 envid 进程造成页缺失时，内核将会把页缺失记录入异常栈 (exceptionstack),然后转向处理函数 'func'。
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
// 分配一页内存在'envid'进程对应的地址空间中，让'va'以'perm'的权限位映射它。新分配的那页内容要清零。如果已有一个va映射到了该页，那么要解映射。
//
// perm -- PTE_U | PTE_P must be set, PTE_AVAIL | PTE_W may or may not be set,
//         but no other bits may be set.  See PTE_SYSCALL in inc/mmu.h.
//
// Return 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
//	-E_INVAL if va >= UTOP, or va is not page-aligned.
//	-E_INVAL if perm is inappropriate (see above).
//	-E_NO_MEM if there's no memory to allocate the new page,
//		or to allocate any necessary page tables.
```
我一开始对这个函数有个小小的疑问，我们为何要辛辛苦苦地使用sys_mem_alloc而不使用我们第二次实验在pmap.c里面的

```C
int^M
write(int fdnum, const void *buf, u_int n)^M
{^M
        int r;^M
        struct Dev *dev;^M
        struct Fd *fd;^M
        //writef("write comes 1\n");^M
        if ((r = fd_lookup(fdnum, &fd)) < 0^M
        ||  (r = dev_lookup(fd->fd_dev_id, &dev)) < 0)^M
                user_panic("write fail at 1.");
//writef("write comes 2\n");^M
        if ((fd->fd_omode & O_ACCMODE) == O_RDONLY) {^M
                writef("[%08x] write %d -- bad mode\n", env->env_id, fdnum);^M
                return -E_INVAL;^M
        }
//writef("write comes 3\n");
        if (debug)
            writef("write %d %p %d via dev %s\n",fdnum, buf, n, dev->dev_name);
        writef("before r:%d\n",r);
        r = (*dev->dev_write)(fd, buf, n, fd->fd_offset);
        writef("after r:%d\n",r);
        if (r > 0)
                fd->fd_offset += r;
//writef("write comes 4\n");
        return 31;^M
}^M

```
