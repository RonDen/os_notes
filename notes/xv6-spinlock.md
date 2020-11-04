```c
// spinlock.h
// 互斥锁，Mutual exclusion lock
struct spinlock
{
  uint locked; // Is the lock held?  是否被占用，0表示没有，1表示已经被占用

  // For debugging:
  char *name;      // Name of lock.  				锁的名称
  struct cpu *cpu; // The cpu holding the lock.  	持有该锁的cpu
  uint pcs[10];    // The call stack (an array of program counters)  调用栈，一个数组记录程序计数器PC
                   // that locked the lock.
};
```

```c
// spinlock.c
void initlock(struct spinlock *lk, char *name)
{
  lk->name = name;
  lk->locked = 0;
  lk->cpu = 0;
}
```

```c
// spinlock.c
// 获取该锁，自旋（循环）直到获取到这个锁
// Acquire the lock.
// Loops (spins) until the lock is acquired.
// Holding a lock for a long time may cause
// other CPUs to waste time spinning to acquire it.
// 长期持有一个锁，可能导致其他的CPU浪费太多的时间来等待（自旋）获取到它
void acquire(struct spinlock *lk)
{
    // 关闭中断，避免死锁现象
  pushcli(); // disable interrupts to avoid deadlock.
    // 如果已经获得了这个锁，还在尝试获取，那么出现系统错误，进入panic
  if (holding(lk))
    panic("acquire");

  // The xchg is atomic.
    // xchg指令是原子性的
  while (xchg(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen after the lock is acquired.
    // 告诉C编译器和处理器不要将负载或存储移过这个点，以确保临界段的内存引用发生在获得锁之后。
  __sync_synchronize();

  // Record info about lock acquisition for debugging.
    // 将关于谁锁占有的信息记录起来，便于调试
  lk->cpu = mycpu();
  getcallerpcs(&lk, lk->pcs);
}
```

```c
// Release the lock.
// 释放该锁
void release(struct spinlock *lk)
{
    // 如果当前不包含锁，错误，进入panic
  if (!holding(lk))
    panic("release");

  lk->pcs[0] = 0;
    // 将spinlock中的cpu设置为0
  lk->cpu = 0;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other cores before the lock is released.
  // Both the C compiler and the hardware may re-order loads and
  // stores; __sync_synchronize() tells them both not to.
    /**
    告诉C编译器和处理器不要移动超过这一点的负载或存储，以确保在锁被释放之前，临界区的所有存储对其他核是可见的。C编译器和硬件都可以重新排序加载和存储;其余的sync_synchronize()告诉它们都不这样做。
    */
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code can't use a C assignment, since it might
  // not be atomic. A real OS would use C atomics here.
  asm volatile("movl $0, %0"
               : "+m"(lk->locked)
               :);
    // 将lk->locked赋值为0，使用指令确保原子操作

  popcli();
    // 开启中断
}
```

```c
// 获取当前调用方的pcs
// 通过下面的ebp链，将当前的调用栈记录在pcs数组中
// Record the current call stack in pcs[] by following the %ebp chain.
void getcallerpcs(void *v, uint pcs[])
{
  uint *ebp;
  int i;

  ebp = (uint *)v - 2;
  for (i = 0; i < 10; i++)
  {
    if (ebp == 0 || ebp < (uint *)KERNBASE || ebp == (uint *)0xffffffff)
      break;
    pcs[i] = ebp[1];      // saved %eip
    ebp = (uint *)ebp[0]; // saved %ebp
  }
  for (; i < 10; i++)
    pcs[i] = 0;
}
```

```c
// Check whether this cpu is holding the lock.
// 检查当前CPU是否持有这个锁
int holding(struct spinlock *lock)
{
  int r;
    // 关中断
  pushcli();
  r = lock->locked && lock->cpu == mycpu();
  popcli();
    // 开中断
  return r;
}
```

```c
// Pushcli/popcli are like cli/sti except that they are matched:
// it takes two popcli to undo two pushcli.  Also, if interrupts
// are off, then pushcli, popcli leaves them off.
/**
Pushcli/popcli与cli/sti相似，只是它们是匹配的:需要两个popcli来撤销两个Pushcli。同样，如果中断是关闭的，那么pushcli, popcli将离开它们。
*/
void pushcli(void)
{
  int eflags;

  eflags = readeflags();
  cli();
  if (mycpu()->ncli == 0)
    mycpu()->intena = eflags & FL_IF;
  mycpu()->ncli += 1;
}

void popcli(void)
{
  if (readeflags() & FL_IF)
    panic("popcli - interruptible");
  if (--mycpu()->ncli < 0)
    panic("popcli");
  if (mycpu()->ncli == 0 && mycpu()->intena)
    sti();
}
```

