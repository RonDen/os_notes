```sh
commit 55e95b16db458b7f9abeca96e541acbdf8d7f85b (HEAD)
Author: rtm <rtm>
Date:   Mon Jun 12 15:22:12 2006 +0000
```

```sh
git checkout 55e95b16db458b7f9abeca96e541acbdf8d7f85b
```

```c
// proc.h
// 进程定义头文件
/*
 * segments in proc->gdt
 */
#define SEG_KCODE 1 // kernel code
#define SEG_KDATA 2 // kernel data+stack
#define SEG_UCODE 3
#define SEG_UDATA 4
#define SEG_TSS 5   // this process's task state
#define NSEGS 6 // 段数量

struct proc{
  char *mem; // start of process's physical memory 用户物理内存的起始位置
  unsigned sz; // total size of mem, including kernel stack 总内存大小，包含内核栈
  char *kstack; // kernel stack, separate from mem so it doesn't move 内核栈，与内存分隔开，不会移动。使用了一个char*来表示栈，真的是太牛了
  enum { UNUSED, RUNNABLE, WAITING } state; // 未使用，运行，等待

  struct Taskstate ts;  // only to give cpu address of kernel stack // in mmu.h
  struct Segdesc gdt[NSEGS]; // 定义在mmu.h内存管理单元文件中
  struct Pseudodesc gdt_pd; // mmu.h
  unsigned esp; // kernel stack pointer  内核栈指针
  unsigned ebp; // kernel frame pointer  内核帧指针

  struct Trapframe *tf; // points into kstack, used to find user regs  在x86.h中，用c语言+汇编混合定义了一些平台相关内容 
  // tf指向kstack内核栈，用来查找用户寄存器
};

extern struct proc proc[];  // 使用extern关键字定义为全局，为什么呢？全局进程表吗
```

```c
// proc.c
struct proc proc[NPROC]; // 进程表，NPROC=64，最大为64

/*
 * set up a process's task state and segment descriptors
 * correctly, given its current size and address in memory.
 * this should be called whenever the latter change.
 * doesn't change the cpu's current segmentation setup.
 正确启动一个进程任务状态和段描述符，给定当前的大小和内存地址，
 在之后的修改中都会被调用，不改变CPU当前的端启动。
 */
void
setupsegs(struct proc *p)
{
  // 初始化任务状态
  memset(&p->ts, 0, sizeof(struct Taskstate));
  p->ts.ts_ss0 = SEG_KDATA << 3; // #define SEG_KDATA 2 内核数据+栈
  p->ts.ts_esp0 = (unsigned)(p->kstack + KSTACKSIZE);

  memset(&p->gdt, 0, sizeof(p->gdt));
  p->gdt[0] = SEG_NULL;
  p->gdt[SEG_KCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, 0);
  p->gdt[SEG_KDATA] = SEG(STA_W, 0, 0xffffffff, 0);
  p->gdt[SEG_TSS] = SEG16(STS_T32A, (unsigned) &p->ts, sizeof(p->ts), 0);
  p->gdt[SEG_TSS].sd_s = 0;
  p->gdt[SEG_UCODE] = SEG(STA_X|STA_R, (unsigned)p->mem, p->sz, 3);
  p->gdt[SEG_UDATA] = SEG(STA_W, (unsigned)p->mem, p->sz, 3);
  p->gdt_pd.pd__garbage = 0;
  p->gdt_pd.pd_lim = sizeof(p->gdt) - 1;
  p->gdt_pd.pd_base = (unsigned) p->gdt;
}

extern void trapret(); // 全局定义，陷入返回

/*
 * internal fork(). does not copy kernel stack; instead,
 * sets up the stack to return as if from system call.
 内部的fork()函数不是拷贝内核中的栈，而是启动一个新的栈
 可以看出，fork函数实际上调用的是这个newproc函数
 */
struct proc *
newproc(struct proc *op)
{
  struct proc *np;
  unsigned *sp;
  // 从第一个进程开始顺序查找，找到第一个状态为UNUSED未使用状态的进程
  // 跳出  
  for(np = &proc[1]; np < &proc[NPROC]; np++)
    if(np->state == UNUSED)
      break;
  if(np >= &proc[NPROC])
    return 0;

  np->sz = op->sz;
  np->mem = kalloc(op->sz);  // 定义在kalloc.c
  if(np->mem == 0)  // 内存分配不成功
    return 0;
  memcpy(np->mem, op->mem, np->sz);  // memcpy定义在string.c中
  np->kstack = kalloc(KSTACKSIZE);  // 初始化新的栈，大小为页大小
  if(np->kstack == 0){
    kfree(np->mem, op->sz);  // 申请栈失败，释放
    return 0;
  }
  // KSTACKSIZE定义在param.h中，大小为PAGE，即页大小
  np->tf = (struct Trapframe *) (np->kstack + KSTACKSIZE - sizeof(struct Trapframe));
  setupsegs(np);   	// 初始化段
  np->state = RUNNABLE;   // 设置状态为运行态
  
  // set up kernel stack to return to user space
  // 启动内核栈，返回用户空间
  *(np->tf) = *(op->tf);
  sp = (unsigned *) np->tf; // 强转为 unsigned 类型
  *(--sp) = (unsigned) &trapret;  // for return from swtch() 从swtch函数返回，下方定义
  *(--sp) = 0;  // previous bp for leave in swtch()
  np->esp = (unsigned) sp;  // kernel stack pointer
  np->ebp = (unsigned) sp;  // kernel frame pointer

  // console printf console.c
  cprintf("esp %x ebp %x mem %x\n", np->esp, np->ebp, np->mem);
    
  return np;
}

/*
 * find a runnable process and switch to it.
 找到一个运行状态的进程切换到它
 */
void
swtch(struct proc *op)
{
  struct proc *np;
  // 无限循环
  while(1){
    for(np = op + 1; np != op; np++){
      if(np == &proc[NPROC])
        np = &proc[0];  // 返回从0开始
      if(np->state == RUNNABLE)
        break;
    }
    if(np->state == RUNNABLE)
      break;
    // idle...
  }
  
  op->ebp = read_ebp();  // x86.h
  op->esp = read_esp();  // x86.h

  // 被调用方的寄存器被保存了吗？
  // XXX callee-saved registers?

  // 能够工作，但是可能不安全。
  // np-ebp 改变栈指针状态，是否正确不太明确
  // this happens to work, but probably isn't safe:
  // it's not clear that np->ebp will evaluate
  // correctly after changing the stack pointer.
  asm volatile("lgdt %0" : : "g" (np->gdt_pd.pd_lim));
  asm volatile("movl %0, %%esp" : : "g" (np->esp));
  asm volatile("movl %0, %%ebp" : : "g" (np->ebp));
}

```

```c
#define NPROC 64
#define PAGE 4096
#define KSTACKSIZE PAGE
```

```c
// mmu.h
/*
 * This file contains definitions for the x86 memory management unit (MMU),
 * including paging- and segmentation-related data structures and constants,
 * the %cr0, %cr4, and %eflags registers, and traps.
 这个文件包含了x86内存管理单元的定义，包括分页，分段相关的数据结构和常量
 %cr0, %cr4和%eflags 寄存器和traps陷入
 */

/*
 *
 *	Part 1.  Paging data structures and constants.
 * 第一部分，分页相关数据结构和常量
 */
// 线性地址la 由如下3部分结构构成
// A linear address 'la' has a three-part structure as follows:
// 10字节一级索引，10字节二级索引，12字节页内偏移
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |      Index     |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \----------- PPN(la) -----------/
// PDX, PTX,PGOFF 和 PPN 宏 分解线性地址如图所示
// The PDX, PTX, PGOFF, and PPN macros decompose linear addresses as shown.
// To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
// use PGADDR(PDX(la), PTX(la), PGOFF(la)).
// 使用PGADDR(PDX(la), PTX(la), PGOFF(la)) 来构造一个线性地址

// page number field of address
#define PPN(la)		(((uintptr_t) (la)) >> PTXSHIFT)
#define VPN(la)		PPN(la)		// used to index into vpt[]

// page directory index
#define PDX(la)		((((uintptr_t) (la)) >> PDXSHIFT) & 0x3FF)
#define VPD(la)		PDX(la)		// used to index into vpd[]

// page table index
#define PTX(la)		((((uintptr_t) (la)) >> PTXSHIFT) & 0x3FF)

// offset in page
#define PGOFF(la)	(((uintptr_t) (la)) & 0xFFF)

// construct linear address from indexes and offset
#define PGADDR(d, t, o)	((void*) ((d) << PDXSHIFT | (t) << PTXSHIFT | (o)))

// Page directory and page table constants.
#define NPDENTRIES	1024		// page directory entries per page directory 每个页目录中的页表目录项
#define NPTENTRIES	1024		// page table entries per page table  每个页表中的表项

#define PGSIZE		4096		// bytes mapped by a page  一页映射的字节数，页面大小
#define PGSHIFT		12		// log2(PGSIZE)  长度？

#define PTSIZE		(PGSIZE*NPTENTRIES) // bytes mapped by a page directory entry  
#define PTSHIFT		22		// log2(PTSIZE)

#define PTXSHIFT	12		// offset of PTX in a linear address  在线性地址空间上的页表项偏移
#define PDXSHIFT	22		// offset of PDX in a linear address  在线性地址空间上的页表目录项偏移

// Page table/directory entry flags.
#define PTE_P		0x001	// Present  		当前的
#define PTE_W		0x002	// Writeable  		可写的
#define PTE_U		0x004	// User      		用户
#define PTE_PWT		0x008	// Write-Through  	写入
#define PTE_PCD		0x010	// Cache-Disable  	不能Cache
#define PTE_A		0x020	// Accessed     	不能访问
#define PTE_D		0x040	// Dirty      		脏
#define PTE_PS		0x080	// Page Size  		页面大小
#define PTE_MBZ		0x180	// Bits must be zero 

// The PTE_AVAIL bits aren't used by the kernel or interpreted by the
// hardware, so user processes are allowed to set them arbitrarily.
#define PTE_AVAIL	0xE00	// Available for software use  可用

// Only flags in PTE_USER may be used in system calls.  只有在PTE_USER中的可以在系统调用中使用
#define PTE_USER	(PTE_AVAIL | PTE_P | PTE_W | PTE_U)

// address in page table entry 页表项中的地址
#define PTE_ADDR(pte)	((physaddr_t) (pte) & ~0xFFF)  

// 控制寄存器标识
// Control Register flags
#define CR0_PE		0x00000001	// Protection Enable 	开启保护
#define CR0_MP		0x00000002	// Monitor coProcessor  监视协处理器
#define CR0_EM		0x00000004	// Emulation			仿真
#define CR0_TS		0x00000008	// Task Switched		任务调度
#define CR0_ET		0x00000010	// Extension Type		扩展类型
#define CR0_NE		0x00000020	// Numeric Errror		数值错误
#define CR0_WP		0x00010000	// Write Protect		写保护
#define CR0_AM		0x00040000	// Alignment Mask		对齐掩码
#define CR0_NW		0x20000000	// Not Writethrough		不能写透
#define CR0_CD		0x40000000	// Cache Disable		取消缓存
#define CR0_PG		0x80000000	// Paging				分页

#define CR4_PCE		0x00000100	// Performance counter enable	启动性能计数器
#define CR4_MCE		0x00000040	// Machine Check Enable			启动机器检查
#define CR4_PSE		0x00000010	// Page Size Extensions			页大小扩展
#define CR4_DE		0x00000008	// Debugging Extensions			调试扩展
#define CR4_TSD		0x00000004	// Time Stamp Disable			时间戳扩展
#define CR4_PVI		0x00000002	// Protected-Mode Virtual Interrupts	虚拟中断保护模式
#define CR4_VME		0x00000001	// V86 Mode Extensions			V86模式扩展

// Eflags register										Eflags寄存器，扩展寄存器
#define FL_CF		0x00000001	// Carry Flag			进位标识
#define FL_PF		0x00000004	// Parity Flag			奇偶校验位
#define FL_AF		0x00000010	// Auxiliary carry Flag	辅助进位标识
#define FL_ZF		0x00000040	// Zero Flag			零标识
#define FL_SF		0x00000080	// Sign Flag			符号标识
#define FL_TF		0x00000100	// Trap Flag			陷入标识
#define FL_IF		0x00000200	// Interrupt Flag		中断标识
#define FL_DF		0x00000400	// Direction Flag		方向标识
#define FL_OF		0x00000800	// Overflow Flag		溢出标识
#define FL_IOPL_MASK	0x00003000	// I/O Privilege Level bitmask	I/O 私有级别位掩码
#define FL_IOPL_0	0x00000000	//   IOPL == 0			
#define FL_IOPL_1	0x00001000	//   IOPL == 1
#define FL_IOPL_2	0x00002000	//   IOPL == 2
#define FL_IOPL_3	0x00003000	//   IOPL == 3
#define FL_NT		0x00004000	// Nested Task
#define FL_RF		0x00010000	// Resume Flag			继续标识
#define FL_VM		0x00020000	// Virtual 8086 mode	虚拟8086模式
#define FL_AC		0x00040000	// Alignment Check		对齐检查
#define FL_VIF		0x00080000	// Virtual Interrupt Flag	虚中断标识
#define FL_VIP		0x00100000	// Virtual Interrupt Pending	虚中断排队
#define FL_ID		0x00200000	// ID flag

// Page fault error codes
#define FEC_PR		0x1	// Page fault caused by protection violation
#define FEC_WR		0x2	// Page fault caused by a write
#define FEC_U		0x4	// Page fault occured while in user mode
```



```c
// types.h
typedef unsigned long long uint64_t;
typedef unsigned int uint32_t;
typedef unsigned short uint16_t;
typedef unsigned char uint8_t;
typedef uint32_t uintptr_t;
typedef uint32_t physaddr_t;
```



```c
// console.c
#include <types.h>
#include <x86.h>
#include "defs.h"

void
cons_putc(int c)
{
  int crtport = 0x3d4; // io port of CGA 彩色图形适配器io端口
  unsigned short *crt = (unsigned short *) 0xB8000; // base of CGA memory
  int ind;

  // cursor position, 16 bits, col + 80*row  光标位置，16 bits, col+80*row
  outb(crtport, 14);
  ind = inb(crtport + 1) << 8;
  outb(crtport, 15);
  ind |= inb(crtport + 1);

  c &= 0xff;

  if(c == '\n'){
    ind -= (ind % 80);
    ind += 80;
  } else {
    c |= 0x0700; // black on white
    crt[ind] = c;
    ind += 1;
  }

  if((ind / 80) >= 24){
    // scroll up
    memcpy(crt, crt + 80, sizeof(crt[0]) * (23 * 80));
    ind -= 80;
    memset(crt + ind, 0, sizeof(crt[0]) * ((24 * 80) - ind));
  }

  outb(crtport, 14);
  outb(crtport + 1, ind >> 8);
  outb(crtport, 15);
  outb(crtport + 1, ind);
}

void
printint(int xx, int base, int sgn)
{
  char buf[16];
  char digits[] = "0123456789ABCDEF";
  int i = 0, neg = 0;
  unsigned int x;
  
  if(sgn && xx < 0){
    neg = 1;
    x = 0 - xx;
  } else {
    x = xx;
  }

  do {
    buf[i++] = digits[x % base];
  } while((x /= base) != 0);
  if(neg)
    buf[i++] = '-';

  while(i > 0){
    i -= 1;
    cons_putc(buf[i]);
  }
}

/*
 * print to the console. only understands %d and %x.
 */
void
cprintf(char *fmt, ...)
{
  int i, state = 0, c;
  unsigned int *ap = (unsigned int *) &fmt + 1;

  for(i = 0; fmt[i]; i++){
    c = fmt[i] & 0xff;
    if(state == 0){
      if(c == '%'){
        state = '%';
      } else {
        cons_putc(c);
      }
    } else if(state == '%'){
      if(c == 'd'){
        printint(*ap, 10, 1);
        ap++;
      } else if(c == 'x'){
        printint(*ap, 16, 0);
        ap++;
      } else if(c == '%'){
        cons_putc(c);
      }
      state = 0;
    }
  }
}

void
panic(char *s)
{
  cprintf(s, 0);
  cprintf("\n", 0);
  while(1)
    ;
}
```

