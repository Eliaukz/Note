---
title: xv6-thread
tags:
  - 6s081
category: 6s081
---
## Uthread: switching between threads

本实验需要完成用户线程的切换。

首先完成thread_switch  这个可以直接参考kernel/switch.S

```c
//user/uthread_switch.S
	.text

	/*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

	.globl thread_switch
thread_switch:
	/* YOUR CODE HERE */
		sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	ret    /* return to ra */
```

然后需要在thread结构体中保存线程切换的上下文context

在线程切换时，只需要switch上下文内容

最后在线程创建时，只需要将
* func保存在ra寄存器中——返回地址，也就是将要执行的函数起始地址
* 将sp指向栈顶，注意栈顶时地址的高处


```c
//user/uthread.c
// Saved registers for kernel context switches.
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context ctx;
};


void 
thread_schedule(void)
{
  //。。。

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->ctx, (uint64)&next_thread->ctx);
  } else
    next_thread = 0;
}

void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->ctx.ra = (uint64)func;
  t->ctx.sp = (uint64)(&t->stack[STACK_SIZE]);

}

```

## Using threads

首先需要声明一把锁
```c
struct entry *table[NBUCKET];
int keys[NKEYS];
int nthread = 1;
pthread_mutex_t lock;            // declare a lock

```

在get和put时都需要获取锁和释放锁。

```c

static 
void put(int key, int value)
{
  pthread_mutex_lock(&lock);  // acquire lock
  
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock);  // release lock
}

static struct entry*
get(int key)
{
  int i = key % NBUCKET;
 pthread_mutex_lock(&lock);  // acquire lock

  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key) break;
  }
  pthread_mutex_unlock(&lock);  // release lock
  return e;
}
```


不要忘记在main中初始化锁（）

## Barrier

最后这个实验是实现一个简单的信号量。

```c

static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);  // acquire lock
  if(++bstate.nthread < nthread) {
      pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  } else {
      bstate.nthread = 0;
      bstate.round++;
      pthread_cond_broadcast(&bstate.barrier_cond);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);  // release lock
}
```

* 每个线程调用barrier时，增加bstate.nthread，然后睡眠该线程
* 每次所有线程都调用barrier时，应该增加bstate.round，然后唤醒所有线程

这里应该调用pthread_cond_wait实现睡眠，因为这个能被唤醒。

