# OSLab4
BUAA OSLab4
##实验概况##
  在开始实验之前，先对实验整体有个大概的了解，这样能让我们更好地进行实验。
  我们本次实验需要补充的内容包括一整套以sys开头的系统调用函数，其中包括了进程间通信需要的一些系统调用如sys_ipc_can_recv等，以及补充完成fork.c函数，当然也不能少填写`syscall_wrap.S`.
##系统调用##
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
