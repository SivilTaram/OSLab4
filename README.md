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

###sys_mem_alloc###
同样，上手先参考一下MIT-JOS的原生注释：
```C
// Allocate a page of memory and map it at 'va' with permission
// 'perm' in the address space of 'envid'.
// The page's contents are set to 0.
// If a page is already mapped at 'va', that page is unmapped as a
// side effect.
// 
// 分配一页内存在'envid'进程对应的地址空间中，让'va'以'perm'的权限位映射它。
// 新分配的那页内容要清零。如果已有一个va映射到了该页，那么要解映射。
//
// perm -- PTE_U | PTE_P must be set, PTE_AVAIL | PTE_W may or may not be set,
//         but no other bits may be set.  See PTE_SYSCALL in inc/mmu.h.
//权限位PTE_U | PTE_P 必须被给予，PTE_AVAIL | PTE_W 给不给都可以，

// Return 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
//	-E_INVAL if va >= UTOP, or va is not page-aligned.
//	-E_INVAL if perm is inappropriate (see above).
//	-E_NO_MEM if there's no memory to allocate the new page,
//		or to allocate any necessary page tables.
//错误种类有：
//如果传入的参数envid不存在的话，或者调用者并没有权限修改envid时，返回-E_BAD_ENV;
//如果va>=UTOP或者va没有对齐时，返回-E_INVAL;
//如果权限位不合适的话，返回-E_INVAL;
//如果没有空闲页用于alloc一个新页或者新页表，返回-E_NO_MEM。
```
在这个函数中我觉得很重要的一点就是要判断关于va是否超过UTOP的问题，在每一个存在传入参数为va的函数中都应当重视这一点。

关于所有权限位的解释与说明，我们可以参考MIT-JOS的注释，可以发现：
```C
#define PTE_P		0x001	// Present
//PTE_P和我们的PTE_V的作用一致，表明一个页表项(或者页目录项)是有效的。

#define PTE_W		0x002	// Writeable
//PTE_W和我们的PTE_R的作用一致，表明该页表项对应的页是用户可写的。

#define PTE_U		0x004	// User
#define PTE_D		0x040	// Dirty
//PTE_D为什么没有定义，难道不需要写回磁盘吗？这两个权限位我一直都抱有怀疑的态度，有点奇怪。

#define PTE_COW		0x800	
//PTE_COW和我们的PTE_COW的作用一样，也是用于copy on write的一个标志位。


下面是我们实验中的权限位的设置
```C
#define PTE_V           0x0200  // Valid bit
#define PTE_R   		   0x0400  // Dirty bit ,'0' means only read ,otherwise make interrupt
#define PTE_COW         0x0001  // Copy On Write
#define PTE_LIBRARY    0x0004  // share memmory
```

很多我们实验中没有见到却定义了的的`PTE_D`,`PTE_UC`,`PTE_G`都没有出现，以上四个权限位是我们贯穿所有实验的最重要的几个权限位。

我们实验中在fork.c中关于PTE_LIBRARY的判断是极其重要的，当然这是后话，稍后再说。

实际上我们在`sys_mem_alloc`中所需要的权限位`PTE_V`是必要的，而`PTE_R`则不是必须的。所以必须在sys_mem_alloc中判断PTE_V，而不需要判断PTE_R.
```C
if((perm & PTE_V) ==0)
            return -E_INVAL;
```
这一句是必须的，否则我们看到在实验中可能会出现隐患错误。

###sys_mem_map###

这个函数用于内存映射，那么该如何映射，继续参考一下MIT-JOS的注释来看：
```C
// Map the page of memory at 'srcva' in srcenvid's address space
// at 'dstva' in dstenvid's address space with permission 'perm'.
// Perm has the same restrictions as in sys_page_alloc, except
// that it also must not grant write access to a read-only
// page.
// 利用给定的权限位Perm建立'srcenvid'地址空间中'srcva'映射的内存页到
//'dstenvid'地址空间中'dstva'虚地址的映射关系。
// Perm有着和sys_mem_alloc一样的限制，除了以下该点：
// 不可以允许只读页能以写的方式访问(即只读不可以写)
//
// Return 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if srcenvid and/or dstenvid doesn't currently exist,
//		or the caller doesn't have permission to change one of them.
//	-E_INVAL if srcva >= UTOP or srcva is not page-aligned,
//		or dstva >= UTOP or dstva is not page-aligned.
//	-E_INVAL is srcva is not mapped in srcenvid's address space.
//	-E_INVAL if perm is inappropriate (see sys_page_alloc).
//	-E_INVAL if (perm & PTE_W), but srcva is read-only in srcenvid's
//		address space.
//	-E_NO_MEM if there's no memory to allocate the new page,
//		or to allocate any necessary page tables.
// 错误种类:
// 如果 srva 没有映射在 srcenvid的地址空间里，这一点可以通过 page_lookup的返回值来确定；
// 如果 perm & PTE_W 为真，但是 srcva 在srcenvid 地址空间内是只读的，则返回-E_INVAL；
```

那么，什么地址是只读的呢？通过阅读在MIT-JOS里的`./inc/memout.h`里面可以看到，所以我们在`sys_mem_map`里所注意的只需要这两点即可。

###sys_mem_unmap###

```C
// Unmap the page of memory at 'va' in the address space of 'envid'.
// If no page is mapped, the function silently succeeds.
//
// 解除envid地址空间内'va'与其对应物理页的映射关系。
// 如果本来就没有映射页,就默默地成功了。
// Return 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
//	-E_INVAL if va >= UTOP, or va is not page-aligned.
// 错误种类前面都有所见过，这里不复述了
```

其实unmap函数本身并不难填，难的是理解其在pgfault中的作用。后面谈到再写吧。

###sys_env_alloc###

终于到这个函数了，这个函数是`fork`之魂，`fork`函数因为有了这个函数才能十分厉害地返回两个返回值，它也是整个系统调用中的核心函数之一。

首先来看其注释，原本是拆开的，现在合起来以便于理解：

```C
// Allocate a new environment.
// Returns envid of new environment, or < 0 on error.  Errors are:
//	-E_NO_FREE_ENV if no free environment is available.
// 产生错误的原因较少，只有当没有空闲进程控制块时才返回非0值。
// Create the new environment with env_alloc(), from kern/env.c.
// It should be left as env_alloc created it, except that
// status is set to ENV_NOT_RUNNABLE, and the register set is copied
// from the current environment -- but tweaked so sys_env_alloc
// will appear to return 0.
// 使用env_alloc()函数来建立一个新的进程控制块
// 它除了要被建立外，还需要设置其状态为ENV_NOT_RUNNABLE
// 还需要使用当前环境来设置其寄存器状态，但是需要调整一些让函数看起来返回0

// install the pgfault upcall to the child
// tweak the register eax of the child,
// thus, the child will look like the return value
// of the the system call is zero.
// 为子进程建立页错误处理函数(调用函数)，调整child的eax寄存器(2号寄存器)
// 因此，子进程的系统调用看起来返回值是0.

// but notice that the return value of the parent
// is the env id of the child
// 但是要注意，父进程的返回值是子进程的ID。
```

我写的关于sys_env_alloc函数如下：

```C
	int sys_env_alloc(void)
	{
        struct Env *child;

1.        if (env_alloc(&child, curenv->env_id) < 0)
                return -E_NO_FREE_ENV;
2.        bcopy(KERNEL_SP - sizeof(struct Trapframe), &child->env_tf, sizeof(struct Trapframe));

3.        child->env_status = ENV_NOT_RUNNABLE;
4.        child->env_pgfault_handler = 0;
5.        child->env_tf.pc = child->env_tf.cp0_epc;

        //tweak register exa of JOS(register v0 of MIPS) to 0 for 0-return
6.        child->env_tf.regs[2] = 0;

7.        return child->env_id;

	}
```

关于这个函数中，我觉得最重要的一点就是理解为何`sys_env_alloc`在不同的进程可以返回两个返回值？

1.所做的功能其实就是申请一个空白的进程块，使用指针child来指向新申请的进程块，且curenv->env_id为其父进程。  
2.将当前的环境中的所有寄存器的状态全部保存在child->env_tf中，这一步相当于为子进程配置了和父进程完全一眼的进程上下文。  
3.在fork结束之前我们不能将子进程状态设置为RUNNABLE，因为我们还将在父进程中为子进程复制一些资源以及处理一些东西。  
4.这里child->env_pgfault_handler其实为0或者为其他均可，因为在子进程实际启动前我们会主动为子进程设置一个页错误处理函数。  
5&6. 5和6搭配才能可以完整地表明为什么在fork函数中使用如下语句：`envid = sys_env_alloc()`时envid会有两个值，一个为0，一个非0。事实上是这样，在fork函数里，在父进程运行到 `envid = sys_env_alloc`这句话时，实际上变成底层语言，是如下的一个过程：
```C
	sys_env_alloc() -> eax(v0)
	eax -> envid
```
我们首先要运行`sys_env_alloc()`函数，其返回值放在了eax寄存器，在mips中称之为v0寄存器，即regs[2]。然后下一步是将eax寄存器中的值赋给envid。
我们再返回来看一下5&6两句，可以发现，一个是设置子进程的pc为child->env_tf.cp0_epc，我们知道此时child->env_tf.cp0_epc实际上和父进程的cp0_epc是一样的，所以实际上当子进程被调度时，子进程运行的第一条指令实际上是代码段中`sys_env_alloc()`返回后的第一条指令，即 `eax -> envid `，又因为我们另一条语句已经设置子进程的eax(v0)寄存器的值为0，所以在子进程中，envid的值是0。  
7. 由于在子进程中返回0，所以为了容易区分两者，在父进程中返回子进程的ID。

###sys_set_env_status###

这个函数实际上没什么难写，就是为envid对应的进程设置相应的状态，这个在父进程的fork中将会被调用设置子进程RUNNABLE状态让子进程参与调度(因为父进程没办法直接操作子进程)。

###sys_set_trapframe###

这个函数就是为envid对应的进程控制块设置进程上下文而已，在我们本次实验中没有用到，在lab6中将会用到。

后面两个系统调用和通信有关，等写完fork之后再叙述。

了解了这个函数，就可以直接讲`fork`的机制了  

##fork##

在讲`fork`的机制之前，首先要谈一下关于函数`pgfault`和函数`duppage`的填写及其作用。  

###pgfault###


```C
// Custom page fault handler - if faulting page is copy-on-write,
// map in our own private writable copy.
// 如果缺页是copy-on-write的，那么则把它复制一份给子进程。(so...)
```  
通过注释的阅读，可以发现`pgfault`实际上就是一个处理页错误时拥有copy-on-write的页的问题的，正常的缺页中断的处理都是有@@@pageout@@@的标识，然后通过tlb进行补页的。那么来细细观察一下`pgfault`的结构：

```C
static void
pgfault(u_int va)
{
        int r;
        int i;
        va = ROUNDDOWN(va, BY2PG);
        
1.      if (!((*vpt)[VPN(va)] & PTE_COW ))
                user_panic("PTE_COW failed!");
                
2.      if (syscall_mem_alloc(0, PFTEMP, PTE_V|PTE_R) < 0)
                user_panic("syscall_mem_alloc failed!");
                
3.      user_bcopy((void*)va, PFTEMP, BY2PG);
        
4.      if (syscall_mem_map(0, PFTEMP, 0, va, PTE_V|PTE_R) < 0)
                user_panic("syscall_mem_map failed!");
                
5.      if (syscall_mem_unmap(0, PFTEMP) < 0)
                user_panic("syscall_mem_unmap failed!");
}
```
首先要搞清楚哪个是父进程的地址空间，哪个又是子进程的地址空间，参考MIT的注释可以得到如下助攻：

```C
	// Allocate a new page, map it at a temporary location (PFTEMP),
	// copy the data from the old page to the new page, then move the new
	// page to the old page's address.
	// Hint:
	//   You should make three system calls.
	//   No need to explicitly delete the old page's mapping.
	// 分配一页，把它映射到一个临时位置(PFTEMP)，把旧页的数据拷到新页去，然后把新页再移到旧页的地址上去。
	// 提示:
	// 你应当使用三个系统调用，无需显式删除旧页的映射。(实际上在page_insert里已经做了解除映射)
```
1. 这句话就是在判断需要处理的页究竟是不是Copy-on-write的，如果不是的话将会报错。这里的`*vpt`是回环搜索，稍后会讲到
2. 在PFTEMP处新申请了一页，并且这里调用系统服务时使用的`envid=0`，这是在`envid2env`中的设定，如果传入的id=0时表示当前进程，所以即是在当前进程的PFTEMP处新申请了一页。  
3. 将va处的一页的内容拷贝到PFTEMP处。  
4. 将PFTEMP处的页作为新的一页(意思就是va旧页的内容作为新页)插入到va所对应的虚拟地址处。  
5. 解除PFTEMP到刚刚va所对应的页的映射，否则下一次用的时候可能会有问题(重叠与覆盖)

其实我们写完就能发现，实际上我们所做的事情是很玄妙的，这时候达到的效果就是只有子进程的`va`可以找到曾经的那个Copy-on-write的页了！而且我们可以看到，不论是在alloc还是在map的时候这页都会加上写权限，并且不会加Copy-on-write。而PFTEMP=Pgfault Temp，实际上用来倒换新旧页，我们想在改变权限的情况下将copy-on-write的那页重新弄到父进程相同的地址并且不能破坏子进程的原先页的属性，所以就巧妙了用了这样的机制。

###duppage###

说到这个函数我真是服了，关于系统调用的那个bug还没有解决，不过这个函数虽然填写很坑，但是其内容还是比较有趣的，而且有一些很厉害的东西一直埋伏着，一直到lab6给了我当头一棒，23333.  

首先来看看`duppage`的注释：
```C
// Map our virtual page pn (address pn*PGSIZE) into the target envid
// at the same virtual address.  If the page is writable or copy-on-write,
// the new mapping must be created copy-on-write, and then our mapping must be
// marked copy-on-write as well.  (Exercise: Why do we need to mark ours
// copy-on-write again if it was already copy-on-write at the beginning of
// this function?)
// 把虚拟页号 pn 映射到目标进程envid 的同样虚拟地址。
//
// Returns: 0 on success, < 0 on error.
// It is also OK to panic on error
```
```C
inline static void
duppage(u_int envid, u_int pn)
{
        int r;
        u_int addr;
        Pte pte;
        u_int perm;

0.      perm = ((*vpt)[pn]) & 0xfff;

1.      if( (perm & PTE_R)!= 0 || (perm & PTE_COW)!= 0)
        {
2.        	if(perm & PTE_LIBRARY) {
                    perm = perm | PTE_V | PTE_R;
                }
                else{
                    perm = perm | PTE_V | PTE_R | PTE_COW;
                }

3.              if(syscall_mem_map(0, pn * BY2PG, envid, pn * BY2PG, perm) == -1)
                        user_panic("duppage failed at 1");

4.              if(syscall_mem_map(0, pn * BY2PG, 0, pn * BY2PG, perm) == -1)
                        user_panic("duppage failed at 2");
        }
        else{
5.              if(syscall_mem_map(0, pn * BY2PG,envid, pn * BY2PG, perm) == -1)
                        user_panic("duppage failed at 3");
        }
}
```
0. 依旧回环搜索QAQ，得到权限位(页框为20位，后12位是权限位)  
1. duppage的含义在于
2. zheyi

`fork`中我写的源码如下：
```C
	int fork(void)
	{
        // Your code here.
        u_int envid;
        int pn;
        extern struct Env *envs;
        extern struct Env *env;

0.      set_pgfault_handler(pgfault);

1.      if((envid = syscall_env_alloc()) < 0)
                user_panic("syscall_env_alloc failed!");

        if(envid == 0)
        {
2.              env = &envs[ENVX(syscall_getenvid())];
                return 0;
        }
3.      for(pn = 0; pn < ( UTOP / BY2PG) - 1 ; pn ++){
4.              if(((*vpd)[pn/PTE2PT]) != 0 && ((*vpt)[pn]) != 0){
5.                      duppage(envid, pn);
                }
        }
6.      if(syscall_mem_alloc(envid, UXSTACKTOP - BY2PG, PTE_V|PTE_R) < 0)
                user_panic("syscall_mem_alloc failed~!");
7.      if(syscall_set_pgfault_handler(envid, __asm_pgfault_handler, UXSTACKTOP) < 0)
                user_panic("syscall_set_pgfault_handler failed~!");
8.      if(syscall_set_env_status(envid, ENV_RUNNABLE) < 0)
                user_panic("syscall_set_env_status failed~!");

        return envid;
	}
```
0. `fork`函数中一开始要为父进程设置页错误处理函数为`pgfault`，这个`pgfault`其实就是上面所填的那个`pgfault`函数。
这里的set_pgfault_handler其参数实际上是一个函数指针，即意味着pgfault是作为函数指针的参数传入set_pgfault_handler函数的。
来观察一下这个函数，就可以知道其作用了：

```C
void
set_pgfault_handler(void (*fn)(u_int va, u_int err))
{
        int r;  
        if (__pgfault_handler == 0) {
                // Your code here:^M
                // map one page of exception stack with top at UXSTACKTOP
                // register assembly handler and stack with operating system
                // 为异常栈(栈顶为UXSTACKTOP)分配一页，在操作系统中注册错误处理函数和栈。
                if(syscall_mem_alloc(0, UXSTACKTOP - BY2PG, PTE_V|PTE_R)<0 || syscall_set_pgfault_handler(0, __asm_pgfault_handler, UXSTACKTOP)<0)
                {
                        writef("cannot set pgfault handler\n");
                        return;
                }
        }
        // Save handler pointer for assembly to call.
        __pgfault_handler = fn;
}
```
这里值得注意的一点就是因为set_pgfault_handler是个用户态的处理函数(因为注册的是用户栈)，所以只能使用syscall开头的系统调用服务。其实能看出，这个函数和系统调用`syscall_set_pgfault_handler`不同之处在于该函数会判断当前的错误处理函数是否为空。所以我们只能对父进程使用该函数，而对子进程一定要新建错误栈并通过系统调用来注册。  

1. 这里的envid<0时出错，很简单，因为正常只会返回0和返回正整数值。
2. 当envid==0时，表明当前 fork函数所在的进程为子进程，所以我们使用`env=&envs[ENVX(syscall_getenvid())];`这里的env可是相当有来历，env是来自外部的`./user/libos.c`里的一个参数。实际上通过`entry.S`我们可以发现，在lab4整个实验中，真正的入口函数应当是从`libmain`开始的，_start叶函数执行完毕后就会跳转到`libmain`执行。`libmain`中实际上是让`env`指向我们当前的进程，然后才开始执行我们所使用的`fktest`或者`pingpong`中的umain来实验。这个`env`的作用是在进程通信的时候使用的。所以这里的这一步也必不可少。
3. 这个地方值得注意的地方在于应该在`pn<USTACKTOP/BY2PG`的地方进行duppage，否则我们将会把父进程的[UXSTACKTOP-BY2PG,UXSTACKTOP] 地址空间同样duppage到子进程的该地址空间上，这个区域是页错误处理的栈区！我们为父进程和子进程分配的错误栈不应该是完全一致的，因为两个进程被调度的时机不同，所以不同时刻进行的程度不一，所以我们应为子进程新分配一个错误栈。  
4. 这里的搜索原理其实挺复杂的，看一篇讲义称其为回环搜索，而我们的数组`vpd[]` 和 `vpt[]` 则是回环搜索的数组，我们首先来看一下这两个数组的作用注释：
```C
/*
 * The page directory entry corresponding to the virtual address range
 * [VPT, VPT + PTSIZE) points to the page directory itself.  Thus, the page
 * directory is treated as a page table as well as a page directory.
 * 虚拟地址[VPT,VPT + PTSIZE) 指向的是页目录自身(即自映射)，因此，页目录就像一个页表一样(查找页表的页表)
 * One result of treating the page directory as a page table is that all PTEs
 * can be accessed through a "virtual page table" at virtual address VPT (to
 * which vpt is set in entry.S).  The PTE for page number N is stored in
 * vpt[N].  (It's worth drawing a diagram of this!)
 * vpt在entry.S里已经被置值为VPT。
 * A second consequence is that the contents of the current page directory
 * will always be available at virtual address (VPT + (VPT >> PGSHIFT)), to
 * which vpd is set in entry.S.
 */
```
