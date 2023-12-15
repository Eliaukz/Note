---
title: xv6-lock
category: 6s081
tags:
  - 6s081
---

## Memory allocator

本实验要求实现一个并发的内存分配策略，在原有的程序中，所有的cpu都向同一个kmem请求资源，导致程序的运行效率低。因此基本思想就是为每一个cpu维护一个kmem，然后分别分配内存。如果某个cpu所需的页面不足，就向其他的cpu的kmem窃取内存页面。

实验要求打印cpu的信息，推荐使用snprintf函数，不过图简单我使用了数组保存cpu信息。

首先修改kmem，对每个cpu维护一个链表
初始化锁的时候要对每一个都初始化。

```c
//kernel/kalloc.c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];

char *kmem_lock_names[] = {
  "kmem_cpu_0",
  "kmem_cpu_1",
  "kmem_cpu_2",
  "kmem_cpu_3",
  "kmem_cpu_4",
  "kmem_cpu_5",
  "kmem_cpu_6",
  "kmem_cpu_7",
};


void
kinit()
{
  //initlock(&kmem.lock, "kmem");
  for (int i = 0; i < NCPU; i++){
      initlock(&kmem[i].lock, kmem_lock_names[i]);
  }
  freerange(end, (void *)PHYSTOP);
}
```


然后修改kfree和kalloc，注意，实验提示在获取cpuid时，需要先关中断，然后操作完再开中断，这样获取的cpuid才是正确的。



```c
//kernel/kalloc.c

// Free the page of physical memory pointed at by v,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off();

  int id = cpuid();
  acquire(&kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&kmem[id].lock);

  pop_off();
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;
  push_off();
  int id = cpuid();
  acquire(&kmem[id].lock);
  
  if(!kmem[id].freelist) { // no page left for this cpu
    int sum = 32; // steal 32 pages from other cpu(s)
    for (int i = 0; i < NCPU; i++) {
        if (i == id) continue;  // no self-robbery
        acquire(&kmem[i].lock);
        struct run *rr = kmem[i].freelist;
        while (rr && sum) {
            kmem[i].freelist = rr->next;
            rr->next = kmem[id].freelist;
            kmem[id].freelist = rr;
            rr = kmem[i].freelist;
            sum--;
        }
        release(&kmem[i].lock);
        if (sum == 0) break;  // done stealing
    }
  }

  r = kmem[id].freelist;

  if(r)
    kmem[id].freelist = r->next;
  release(&kmem[id].lock);
  
  pop_off();
  if (r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

```


在kalloc函数中，除了正常分配页表外，还需要检测当前分配的页表链表是否为空，如果为空，需要从别的链表中窃取页表，这里我选择的数量是32 ，














