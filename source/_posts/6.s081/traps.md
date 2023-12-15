---
title: xv6-traps
category: 6s081
tags:
  - 6s081
---
## Backtrace

要求我们在printf函数中实现backtrace函数

按照提示，首先在riscv.h中添加读取fp寄存器的代码。

然后就可以着手实现backtrace的代码了

```c
//kernel/printf.c
void backtrace(void) {
    uint64 fp = r_fp();
    uint64 *frame = (uint64 *)fp;
    uint64 up = PGROUNDUP(fp);
    uint64 down = PGROUNDDOWN(fp);
    while (fp < up && fp > down) {
        printf("%p\n", frame[-1]);
        fp = frame[-2];
        frame = (uint64 *)fp;
    }
}
```

查看栈的结构可知，栈是一个一维的数组，然后fp地址的上一个地址就是ra--返回地址。
不妨将地址转换为数组，这样去下标为-1就是上一个ra的地址，取-2就是上一个fp的地址。
（这是我参照大佬的blog学习来的）

然后在def.h中加入函数原型，然后在sys_sleep中调用该函数即可完成实验。

## Alarm

本实验需要实现两个系统调用，添加系统调用的过程不在赘述，可参照syscall实验完成。


然后需要从当前函数中断跳转到handler函数中，所以需要保存trapframe信息，以及tick的次数等信息，所以这些信息需要被保存在proc结构体中

```c
//kernel/proc.h

// Per-process state
struct proc {
  
  char name[16];               // Process name (debugging)
  int ticks;
  int ticks_cnt;
  uint64 handler;
  struct trapframe tick_trapframe;
  int tick_handled;
};

```

然后接可以实现sys_sigalarm函数，即从用户态程序获取参数保存在proc结构体中，sys_sigreturn函数就是恢复原来的函数继续执行

```c
//kernel/sysproc.c

uint64 sys_sigalarm(void) {
    int ticks;
    uint64 handler;
    argint(0, &ticks);
    argaddr(1, &handler);

    struct proc* p = myproc();
    p->ticks = ticks;
    p->handler = handler;
    return 0;
}

uint64 sys_sigreturn(void) {
    struct proc* p = myproc();
    *(p->trapframe) = p->tick_trapframe;
    p->tick_handled = 0;
    return 0;
}

```

最后就是处理时钟中断，放弃cpu前的操作，

```c
//kernel/trap.c
void
usertrap(void)
{
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2){
    if (p->ticks > 0) {
        p->ticks_cnt++;
        if (p->ticks_cnt > p->ticks && !p->tick_handled) {
            p->tick_handled = 1;
            p->ticks_cnt = 0;
            p->tick_trapframe = *(p->trapframe);
            p->trapframe->epc = p->handler;
        }
    }
    yield();
  }
}
```



