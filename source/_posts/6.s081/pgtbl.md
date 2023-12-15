---
title: xv6-pgtbl
tags:
  - 6s081
category: 6s081
---
## Speed up system calls

本实验需要在内核与用户控件共享数据，因此需要在页表的只读空间增加一个映射，然后将需要共享的信息保存到这个映射处。

按照提示，首先需要在kernel/proc.c中的proc_pagetable()中执行映射。
查看allocproc函数，调用kalloc分配一个页表。

因此需要首先在proc结构体中新增一个数据段。
```c
//kernel/proc.h

// Per-process state
struct proc {
  //...
  struct trapframe *trapframe; // data page for trampoline.S
  struct usyscall *usyscall;    // usycall    页表
  struct context context;      // swtch() here to run process
  //...
};
```

在分配页表之后，需要对页表进行map，使新的页表能够保存usyscall信息。

```c
//kernel/proc.c
pagetable_t
proc_pagetable(struct proc *p)
{
//...
  // map the trapframe page just below the trampoline page, for
  // trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

    if (mappages(pagetable, USYSCALL, PGSIZE,
               (uint64)(p->usyscall), PTE_R | PTE_U) < 0) {
      uvmunmap(pagetable, TRAPFRAME, 1, 0);
      uvmunmap(pagetable, TRAMPOLINE, 1, 0);
      uvmfree(pagetable, 0);
      return 0;
  }

  return pagetable;
}
```


然后初始化usyscall，当然在free的时候不要忘记释放这个新分配的页表


```c
//kernel/proc.c
static struct proc*
allocproc(void)
{
// Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  //初始化usyscall
  if ((p->usyscall = (struct usyscall *)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }

}


static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  if(p->usyscall)
      kfree((void *)p->usyscall);
  p->usyscall = 0;
  
}

```


然后运行 `make qemu` 会发现无法启动

```bash
xv6 kernel is booting

hart 1 starting
hart 2 starting
panic: freewalk: leaf
```

是因为我们在释放页表是，没有unmap usyscall这个页表

```c
//kernel/peoc.c
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0);
  uvmfree(pagetable, sz);
}

```


## Print a page table

这个任务比较简单，

首先在exec.c中添加

```c
//kernel/exec.c
int
exec(char *path, char **argv)
{
  proc_freepagetable(oldpagetable, oldsz);
  if (p->pid == 1) vmprint(p->pagetable);
  return argc;  // this ends up in a0, the first argument to main(argc, argv)

}
```

然后在vm.c中实现vmprint函数

此函数的实现主要参照walk函数的实现

```c
//kernel/vm.c
void _vmprint(pagetable_t p, int depth) {
  if (depth > 3) return;
  for (int i = 0; i < 512; i++) {
      pte_t pte = p[i];
      if (pte & PTE_V) {
          for (int j = 0; j < depth; j++) printf(" ..");
          uint64 child = PTE2PA(pte);
          printf("%d: pte %p pa %p\n", i, pte, child);
          _vmprint((pagetable_t)child, depth + 1);
      }
    }
}

void vmprint(pagetable_t p) {
  printf("page table %p\n", p);
  _vmprint(p, 1);
}
```

最后在def.h中添加函数声明

这里讲解一下PTE2PA 
因为页表框的右十位是标记位 ，因此需要删除，而物理地址是物理页号+页内偏移，每页的大小是4kb，因此，需要将右12位作为偏移，所以需要左移12位。

## Detect which pages have been accessed

首先要查看PTE_A的标记位，这里需要查看riscv手册，在cpu访问到某页面时，会自动将该位置为1。
查阅手册可知，是第六位，因此需要在riscv.h中添加标记

```c
//kernel/riscv.h

#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // user can access
#define PTE_A (1L << 6) // access 
```

然后完成sys_pgaccess函数

```c
//kernel/sysproc.h
#ifdef LAB_PGTBL
int sys_pgaccess(void) {
    // lab pgtbl: your code here.
    uint64 addr;
    int n;
    int bitmask;
    if (argaddr(0, &addr) < 0 || argint(1, &n) < 0 || argint(2, &bitmask) < 0)
        return -1;
    if (n > 32 || n < 0)
        return -1;

    struct proc* p = myproc();
    int res = 0;

    for (int i = 0; i < n; ++i) {
        pte_t* pte = walk(p->pagetable, addr + i * PGSIZE, 0);
        if (pte == 0) continue;
        if (*pte & PTE_A) {
            res |= (1 << i);
            *pte &= ~PTE_A;
        }
    }

    if (copyout(p->pagetable, bitmask, (char*)&res, sizeof(res)) < 0)
        return -1;
    return 0;
}
#endif

```

如果访问过，就将对应的位置为1，然后将PTE_A置为0

最后使用copyout将得到的结果返回给用户态。

## summary
在第一个实验中提到了加速系统调用，至于可以加速哪些系统调用，我查阅资料后得知，例如fork系统调用，在usyscall结构体中，添加`struct proc *parent`  保存该进程的父进程信息，然后在fork时不需要去查阅该进程的父进程就可以加快fork系统调用。








