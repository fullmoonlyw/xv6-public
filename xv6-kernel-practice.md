xv6 Kernel 学习笔记

学生信息

\- 姓名：吴艳

\- 班级：23级计科1

\- 实践时间：2026年春季

\- 视频链接：https://www.bilibili.com/video/BV1w94y1a7i8/

一、系统调用（System Calls）

1.1 系统调用执行流程



用户程序调用write() → user/usys.S（ecall陷入内核） → kernel/trampoline.S → kernel/trap.c（usertrap） → kernel/syscall.c（syscall分发） → kernel/sysfile.c（sys\_write） → 返回用户态



1.2 关键代码位置



| 文件 | 作用 |

|------|------|

| user/user.h | 用户态函数声明 |

| user/usys.S | ecall汇编入口 |

| kernel/syscall.h | 系统调用号定义 |

| kernel/syscall.c | syscall()分发函数 |

| kernel/sysproc.c | 进程相关系统调用 |

| kernel/sysfile.c | 文件相关系统调用 |

1.3 核心代码分析



\*\*syscall()分发函数\*\*（kernel/syscall.c）：

```c

void syscall(void) {

&#x20; int num = p->trapframe->a7;  // 获取系统调用号

&#x20; if(num > 0 \&\& num < NELEM(syscalls) \&\& syscalls\[num]) {

&#x20;   p->trapframe->a0 = syscalls\[num]();  // 调用对应函数

&#x20; } else {

&#x20;   printf("%d %s: unknown sys call %d\\n", p->pid, p->name, num);

&#x20;   p->trapframe->a0 = -1;

&#x20; }

}

系统调用号定义（kernel/syscall.h）：

\#define SYS\_fork   1

\#define SYS\_exit   2

\#define SYS\_wait   3

\#define SYS\_pipe   4

\#define SYS\_read   5

\#define SYS\_kill   6

\#define SYS\_exec   7

\#define SYS\_fstat  8

\#define SYS\_chdir  9

\#define SYS\_dup    10

\#define SYS\_getpid 11

\#define SYS\_sbrk   12

\#define SYS\_sleep  13

\#define SYS\_uptime 14

\#define SYS\_open   15

\#define SYS\_write  16

\#define SYS\_mknod  17

\#define SYS\_unlink 18

\#define SYS\_link   19

\#define SYS\_mkdir  20

\#define SYS\_close  21

1.4 新增系统调用步骤

以添加hello()系统调用为例：

第1步：user/user.h 添加声明

int hello(void);

第2步：user/usys.pl 添加entry

perl

entry("hello");

第3步：kernel/syscall.h 添加编号

\#define SYS\_hello 22

第4步：kernel/syscall.c 添加跳转

extern uint64 sys\_hello(void);

// 在syscalls数组中添加

\[SYS\_hello] sys\_hello,

第5步：kernel/sysproc.c 实现

uint64 sys\_hello(void) {

&#x20;   return 100;  // 返回固定值

}

第6步：用户程序测试

int main() {

&#x20;   printf("hello() = %d\\n", hello());

&#x20;   exit(0);

}

二、同步机制（Spinlock / Sleep \& Wakeup）

2.1 spinlock（自旋锁）

数据结构（kernel/spinlock.h）：

struct spinlock {

&#x20;   uint locked;       // 是否被持有，1=已锁，0=未锁

&#x20;   char \*name;        // 锁名称（调试用）

&#x20;   struct cpu \*cpu;   // 持有锁的CPU

};

acquire获取锁（kernel/spinlock.c）：

void acquire(struct spinlock \*lk) {

&#x20;   push\_off();  // 关闭中断

&#x20;   if(holding(lk))

&#x20;       panic("acquire");

&#x20;   // 原子操作：测试并设置

&#x20;   while(\_\_sync\_lock\_test\_and\_set(\&lk->locked, 1) != 0)

&#x20;       ;  // 循环等待（自旋）

&#x20;   \_\_sync\_synchronize();  // 内存屏障

&#x20;   lk->cpu = mycpu();

}

release释放锁：

void release(struct spinlock \*lk) {

&#x20;   if(!holding(lk))

&#x20;       panic("release");

&#x20;   lk->cpu = 0;

&#x20;   \_\_sync\_synchronize();

&#x20;   \_\_sync\_lock\_release(\&lk->locked);  // 原子清零

&#x20;   pop\_off();  // 恢复中断

}

使用示例：

struct spinlock lock;

initlock(\&lock, "my\_lock");

acquire(\&lock);

// 临界区代码

release(\&lock);

2.2 sleep \& wakeup（睡眠与唤醒）

核心思想：让出CPU，避免忙等待

sleep函数：

void sleep(void \*chan, struct spinlock \*lk) {

&#x20;   if(lk != \&p->lock) {

&#x20;       acquire(\&p->lock);

&#x20;       release(lk);

&#x20;   }

&#x20;   p->chan = chan;

&#x20;   p->state = SLEEPING;

&#x20;   sched();

&#x20;   p->chan = 0;

&#x20;   if(lk != \&p->lock) {

&#x20;       release(\&p->lock);

&#x20;       acquire(lk);

&#x20;   }

}

wakeup函数：

void wakeup(void \*chan) {

&#x20;   for(p = proc; p < \&proc\[NPROC]; p++) {

&#x20;       if(p->state == SLEEPING \&\& p->chan == chan) {

&#x20;           p->state = RUNNABLE;

&#x20;       }

&#x20;   }

}

生产者-消费者示例：

// 生产者

acquire(\&lock);

while(buffer\_full()) {

&#x20;   sleep(\&buffer, \&lock);

}

produce\_item();

wakeup(\&buffer);

release(\&lock);



// 消费者

acquire(\&lock);

while(buffer\_empty()) {

&#x20;   sleep(\&buffer, \&lock);

}

consume\_item();

wakeup(\&buffer);

release(\&lock);

2.3 sleeplock（可睡眠锁）

特点：持有锁期间可以睡眠，适用于较长临界区

数据结构（kernel/sleeplock.h）：

struct sleeplock {

&#x20;   uint locked;

&#x20;   struct spinlock lk;

&#x20;   char \*name;

&#x20;   int pid;

};

对操作系统设计的理解

通过本次xv6 Kernel系列视频的学习，我对操作系统内核有了更深入的理解：

分层设计：xv6采用清晰的分层架构，用户程序通过系统调用接口与内核交互，内核服务再通过硬件抽象层访问硬件，每一层职责明确。

资源抽象：操作系统将CPU抽象为进程，内存抽象为地址空间，磁盘抽象为文件，这种抽象让用户程序无需关心底层硬件细节。

隔离与共享：通过页表实现进程间内存隔离，通过文件系统实现数据共享，通过锁机制保护共享资源。

