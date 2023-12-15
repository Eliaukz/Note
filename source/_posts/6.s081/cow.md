---
title: xv6-cow
tags:
  - 6s081
category: 6s081
---
## cow

本实验需要实现一个写时复制的任务。

首先查看手册发现页表的标记位中的9，10位是未使用的标记位，因此我们使用第9位作为copy时的标记位。

在riscv.h中添加

```c
//kernel/riscv.h
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // user can access
#define PTE_COW (1L << 8)  // cow标识位
```


实验要求能够记录每个页表的引用次数，只有当该页表的引用次数为0时，才可以释放。
因此，我们需要记录页表的引用次数，在kalloc.c中实现。

首先修改kmem的结构体内容，首先要记录页表的数量，然后使用数组存储每个页表的引用次数。

参照freerange函数，即可计算出页表的数量。这一步应该在init函数开始时就计算。
然后为了有存储空间能够保存页表的引用数量，我将ref_page赋值为end，然后用kmem.end_替代全局变量end，这样就有空间来存储页表的引用数量。

此外还需要几个辅助函数，
*  page_index 计算给定地址所属的页表的下标 。只需要先取到页表的基地址然后减去出初始地址最后除以PGSIZE
* incr 递增页表的引用计数
* desc 递减页表的引用计数

```c
//kernel/kalloc.c

struct {
  struct spinlock lock;
  struct run *freelist;
  char *ref_page;
  int cnt_mem;
  char *end_;
} kmem;

int pagecnt(void *pa_start, void *pa_end){
  char *p;
  int res = 0;
  p = (char *)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
      res++;
  return res;
}

void 
kinit() {
  initlock(&kmem.lock, "kmem");
  kmem.cnt_mem = pagecnt(end, (void *)PHYSTOP);
  // printf("%p\n", end);
  kmem.ref_page = end;
  for (int i = 0; i < kmem.cnt_mem;i++){
      kmem.ref_page[i] = 0;
  }
  kmem.end_ = kmem.ref_page + kmem.cnt_mem;
  
  freerange(kmem.end_, (void *)PHYSTOP);
}

int page_index(uint64 pa){
  pa = PGROUNDDOWN(pa);
  
  int res = (pa - (uint64)kmem.end_) / PGSIZE;

  if (res < 0 || res >= kmem.cnt_mem) {
      printf("%p\n%p\n", pa, kmem.end_);
      printf("%d\n%d\n%d\n", res, pa, (uint64)kmem.end_);
  }
  return res;
}

void incr(uint64 pa){
  int index = page_index(pa);
  acquire(&kmem.lock);
  kmem.ref_page[index]++;
  release(&kmem.lock);
}

void desc(uint64 pa){
  int index = page_index(pa);
  acquire(&kmem.lock);
  kmem.ref_page[index]--;
  release(&kmem.lock);
}

```

注意要在def.h中增加函数声明。

这时可以先实现kfree函数。

每次调用时都先递减引用计数，如果引用计数等于0，就free掉页面。
在分配一个页面时需要递增引用

```c
//kernel/kalloc.c
void
kfree(void *pa)
{
  int index = page_index((uint64)pa);
  if(kmem.ref_page[index] > 1){
    desc((uint64)pa);
    return;
  }
  if(kmem.ref_page[index] == 1){
    desc((uint64)pa);
  }
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}


void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r){
    memset((char*)r, 5, PGSIZE); // fill with junk
    incr((uint64)r);
  }
  return (void *)r;
}
```


这时可以实现uvmcopy函数。在调用的时候不新分配一个页面，而是将父进程的页面也map给子进程，修改标记位，以节省页表的数量。

```c
//kernel/vm.c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    *pte = *pte & ~PTE_W;
    *pte = *pte | (PTE_COW);
    flags = PTE_FLAGS(*pte);
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }
    incr(pa);
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```


除此之外，还需要实现检测cow的函数，因为copy on write 在程序修改页表时需要为进程复制一个页表，而不能直接在共享的页表上修改。

在检测到页表是cow错误时，需要新分配一个页表。
分配页表函数可以参照修改前的uvmcopy函数实现。

```c
//kernel/vm.c
int 
is_cow_falut(pagetable_t pagetable,uint64 va){
  if(va >= MAXVA){
    return 0;
  }
  va = PGROUNDDOWN(va);
  pte_t *pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  if((*pte & PTE_U) == 0)
    return 0;
  if (*pte & PTE_COW) 
    return 1;
  return 0;
}

int
cow_alloc(pagetable_t pagetable,uint64 va){
  va = PGROUNDDOWN(va);
  pte_t *pte = walk(pagetable, va, 0);
  uint64 pa = PTE2PA(*pte);
  int flag = PTE_FLAGS(*pte);

  char *mem = kalloc();
  if (mem == 0)
    return -1;

  memmove(mem, (char *)pa, PGSIZE);
  uvmunmap(pagetable, va, 1, 1);

  flag &= (~PTE_COW);
  flag |= PTE_W;

  if(mappages(pagetable, va, PGSIZE, (uint64)mem, flag) != 0){
    kfree(mem);
    return -1;
  }
  return 0;
}
```


最后就是检测页表错误，然后分配页表。

```c
//kernel/trap.c
//
void
usertrap(void)
{
  ///...
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if( r_scause() == 13 || r_scause() == 15 ) {
    uint64 va = r_stval();
    if (is_cow_falut(p->pagetable, va )) {
      if (cow_alloc(p->pagetable, va) < 0) {
          p->killed = 1;
      }
    } else p->killed = 1;
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  //...
}

```


在发生中断是，检测是否因为cow导致的中断，如果是就分配新页面，分配失败要将进称kill以免造成错误。

按照实验说明，还需要修改copyout，检测是否cow导致错误。

```c
//kernel/vm.c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if (is_cow_falut(pagetable, va0)) {
      if (cow_alloc(pagetable, va0) < 0) {
        return -1;
      }
    }

    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}

// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

 
  while(len > 0){
    va0 = PGROUNDDOWN(srcva);
    if (is_cow_falut(pagetable, va0)) {
      if (cow_alloc(pagetable, va0) < 0) {
        return -1;
      }
    }
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```


*usertests测试通过不了的话 ，试试make clean一下*

