---
title: xv6-syscall
tags:
  - 6s081
category: 6s081
---
## 系统调用

指运行在用户空间的程序向操作系统内核请求需要更高权限运行的服务。系统调用提供用户程序与操作系统之间的接口。大多数系统交互式操作需求在内核态执行。如设备IO操作或者进程间通信。

xv6实现系统调用的流程

1. 添加函数原型声明到user/user.h,
2. 添加a stub到user/usys.pl
3. 在kernel/syscall.h添加系统调用号
4. 在kernel/syscall.c中添加系统调用
5. 在kernel/sysproc.c    or     kernel/sysfile.c          实现系统调用的函数体


 **前置知识**

系统调用前，处于用户态，系统调用后，处于内核态，因此无法直接传递参数
参数传递需要使用argint / argaddr / argstr  函数来传递参数

### systrace

添加系统调用号

```c
//kernel/syscall.h
#define SYS_trace 22
#define SYS_sysinfo 23
```


因为要保存系统调用信息，因此需要在proc结构体中保存系统调用信息

```c
//kernel/proc.h

// Per-process state
struct proc {
    //...
    struct inode *cwd;            // Current directory
    char name[16];                // Process name (debugging)
    uint64 mark;                  // 标记trace系统调用的标记位
};

```


syscall.c函数需要添加函数名，以及对系统调用的比较，
```c
//kernel/syscall.c
static uint64 (*syscalls[])(void) = {
    [SYS_fork] sys_fork,
    [SYS_exit] sys_exit,
    [SYS_wait] sys_wait,
    [SYS_pipe] sys_pipe,
    [SYS_read] sys_read,
    [SYS_kill] sys_kill,
    [SYS_exec] sys_exec,
    [SYS_fstat] sys_fstat,
    [SYS_chdir] sys_chdir,
    [SYS_dup] sys_dup,
    [SYS_getpid] sys_getpid,
    [SYS_sbrk] sys_sbrk,
    [SYS_sleep] sys_sleep,
    [SYS_uptime] sys_uptime,
    [SYS_open] sys_open,
    [SYS_write] sys_write,
    [SYS_mknod] sys_mknod,
    [SYS_unlink] sys_unlink,
    [SYS_link] sys_link,
    [SYS_mkdir] sys_mkdir,
    [SYS_close] sys_close,
    [SYS_trace] sys_trace,
    [SYS_sysinfo] sys_sysinfo,
};

char *trace_name[] = {
    [SYS_fork] "fork",
    [SYS_exit] "exit",
    [SYS_wait] "wait",
    [SYS_pipe] "pipe",
    [SYS_read] "read",
    [SYS_kill] "kill",
    [SYS_exec] "exec",
    [SYS_fstat] "fstat",
    [SYS_chdir] "chdir",
    [SYS_dup] "dup",
    [SYS_getpid] "getpid",
    [SYS_sbrk] "sbrk",
    [SYS_sleep] "sleep",
    [SYS_uptime] "uptime",
    [SYS_open] "open",
    [SYS_write] "write",
    [SYS_mknod] "mknod",
    [SYS_unlink] "unlink",
    [SYS_link] "link",
    [SYS_mkdir] "mkdir",
    [SYS_close] "close",
    [SYS_trace] "trace",
    [SYS_sysinfo] "sysinfo",
};

void syscall(void) {
    int num;
    struct proc *p = myproc();

    num = p->trapframe->a7;
    if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();
        if (1 << num & p->mark) {
            printf("%d: syscall %s -> %d\n", p->pid, trace_name[num], p->trapframe->a0);
        }
    } else {
        printf("%d %s: unknown sys call %d\n",
               p->pid, p->name, num);
        p->trapframe->a0 = -1;
    }
}

```


同时在fork时需要拷贝父进程的标记位到子进程
```c
//kernel/proc.c

// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int fork(void) {
	//...
    safestrcpy(np->name, p->name, sizeof(p->name));

    np->mark = p->mark;  // 拷贝trace标识位

    pid = np->pid;

    release(&np->lock);

    return pid;
}

```


sys_trace函数实现

```c
uint64
sys_trace(void) {
    uint64 mark;
    if (argaddr(0, &mark) < 0)
        return -1;
    struct proc *p = myproc();
    p->mark = mark;
    return 0;
}

```



### sysinfo


要计算sysinfo信息，则需要计算空闲内存数量和未使用的进程数量

首先在sysproc.c中写出函数原型


```c
//kernel/sysproc.c
uint64
sys_sysinfo(void) {
    uint64 addr;
    if (argaddr(0, &addr) < 0)
        return -1;
    struct sysinfo info;
    info.freemem = kmemory();
    info.nproc = kproc();
    struct proc *p = myproc();
    if (copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
        return -1;

    return 0;
}
```


然后就以此实现计算空闲内存数量的函数kmemory 和计算进程的函数kproc

#### kmemory


在kernel/kalloc.c中实现memory

xv6中的空闲内存页存储在kmem.freelist中，因此只要遍历这个单链表就可以计算得到空闲内存的大小（kmem存的是页表，因此每次需要加一个PGSIZE）

```c
//kernel/kalloc.c
uint64 kmemory(void) {
    struct run *r;
    uint64 ans = 0;
    acquire(&kmem.lock);
    for (r = kmem.freelist; r != 0; r = r->next)
        ans += PGSIZE;
    release(&kmem.lock);
    return ans;
}
```


#### kproc

kproc函数的实现可以参照`kill` or `walk`函数实现


```c
//kernel/proc.c
uint64 kproc(void) {
    uint64 ans = 0;
    struct proc *p;
    for (p = proc; p < &proc[NPROC]; p++) {
        acquire(&p->lock);
        if (p->state != UNUSED)
            ans++;
        release(&p->lock);
    }
    return ans;
}
```


**这里的函数均需要在`kernel/def.h`中声明


然后使用copyout将信息从内核态拷贝到用户态就完成了sysinfo系统调用




