# OSLab4
BUAA OSLab4

##fork##
fork这个函数很奇特，因为我不知道怎么填。
最初下手的时候，感觉是奇妙的，所以至今都不知道下面所写的源码究竟是对还是不对
```C
extern void __asm_pgfault_handler(void);
int
fork(void)
{
        // Your code here.
        u_int envid;
        u_int myid=syscall_getenvid();
        int pagenum;
        int pn,j,r;
        u_int addr;
        extern struct Env *envs;
        extern struct Env *env;

        set_pgfault_handler(pgfault);
        //Step1:set pgfault handler.
        writef("set_pgfault_handler success!\n");


        if((envid=syscall_env_alloc())<0)
                user_panic("syscall_env_alloc error!\n");
        
        writef("This is a father process.");
        
        if(envid == 0){
                env = &envs[ENVX(syscall_getenvid())];
                writef("env_run a new process");
                return 0;
        }
        for(addr=0;addr<UXSTACKTOP-BY2PG;addr+=BY2PG){
                if((*vpd)[PDX(addr)] && (*vpt)[VPN(addr)])
                        duppage(envid,VPN(addr));
        }
        writef("duppage success...\n");
        
        if((r=syscall_mem_alloc(envid,UXSTACKTOP-BY2PG,PTE_V|PTE_R))<0)
                user_panic("envid's UXSTACKTOP syscall_mem_alloc error!\n");
        if((r=syscall_mem_map(envid,UXSTACKTOP-BY2PG,myid,PFTEMP,0))<0)
                user_panic("child syscall_mem_map failed!\n");
        user_bcopy(UXSTACKTOP-BY2PG,PFTEMP,BY2PG);
        if((r=syscall_mem_unmap(myid,PFTEMP))<0)
                user_panic("PFTEMP unmap failed!\n");
        syscall_set_pgfault_handler(envid,__asm_pgfault_handler,UXSTACKTOP);
        syscall_set_env_status(envid,ENV_RUNNABLE);
        
        return envid;

}


```
