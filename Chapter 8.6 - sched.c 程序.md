# Chapter 8.6 - sched.c 程序

Created by : Mr Dk.

2019 / 08 / 16 17:05

Ningbo, Zhejiang, China

---

## 8.6 sched.c 程序

### 8.6.1 功能描述

内核中有关任务 (进程) 调度管理的程序，包含：

* 几个有关调度的基本函数
* 一些简单的系统调用
* 系统时钟中断的定时函数和软盘驱动器定时程序

#### 8.6.1.1 调度函数

调度函数 `schedule()` 负责选择系统中下一个要运行的任务

首先，对所有任务进行检测，__唤醒任何一个已经得到信号的任务__

* 检查报警器定时值 alarm，如果已经过时，就在信号位图中设置 `SIGALRM` 信号，清除 alarm
* 如果进程的信号位图中，除去被阻塞的信号外还有其它信号，并处于可中断睡眠状态 `TASK_RUNNING`，则置任务为就绪状态 `TASK_RUNNING`

随后是调度的核心部分

根据进程的 __时间片__ 和 __优先级__

选择随后要执行的任务

并利用 `switch_to()` 切换到该任务

若所有就绪态任务的时间片都为 0

则根据任务的优先级重新设置每个任务的运行时间片

再次重新检查所有任务的时间片，并进行选择

#### 8.6.1.2 睡眠和唤醒函数

`sleep_on()` 函数的主要功能：

* 当一个进程所请求的资源 __正被占用__ 或 __不在内存__
* 暂时将该进程切换出去，放在等待队列中等待一段时间

函数涉及三个任务指针的操作：

* `*p` - 等待队列头指针
  * 文件系统中 `i_wait` 指针
  * 内存缓冲中 `buffer_wait` 指针
  * ...
* `tmp` - 存储在当前任务的内核态堆栈上，指向前一个正在等待的任务
* `current` - 当前任务指针

`sleep_on()` 函数使 `*p` 指向当前任务，使当前任务的 `tmp` 指针指向 `*p` 原来指向的正在等待的任务

当几个进程为了等待同一资源而分别调用 `sleep_on()` 时

构筑出了一个等待队列：

![8-7](./img/8-7.png)

> 这队列很奇怪 因为它好像不是 FIFO 的.. 😥

将进程插入队列后，`sleep_on()` 函数就会调用 `schedule()` 函数去执行别的进程

当进程被唤醒时，就会把比它更早进入队列的进程唤醒

唤醒函数 `wake_up()` 用于把等待可用资源的指定任务置位就绪状态

`sleep_on()` 还有一种形式 - `interruptible_sleep_on()` 函数

* 调度其它任务前，将当前任务置为 __可中断等待状态__
* 在本任务被唤醒后，还需要判断队列上是否有后来的任务；若有，则需要先调度它们
* Linux 0.12 中，这两种情况都由 `sleep_on()` 实现，用任务的状态作为参数区分这两种情况

### 8.6.2 代码注释

> 进程调度和信号处理之间联系较多
>
> 所以我觉得要结合 `sys_call.s` 和 `signal.c` 一块儿看
>
> 才能加强理解

首先定义了两个宏，用于快速操作信号：

* 信号的编号是从 1 开始，到 32 为止
* 但信号在 bitmap 中是从第 0 位到第 31 位，所以要注意这个转变

```c
// 取编号为 nr 的信号在 bitmap 中的对应数值
#define _S(nr) (1 << ((nr)-1))
// 除 SIGKILL 和 SIGSTOP 信号以外，其它信号是可以被阻塞的
#define _BLOCKABLE (~(_S(SIGKILL) | _S(SIGSTOP)))
```

> 关于信号和阻塞的一点理解：
>
> 内核通过在进程 PCB 中设置信号位来给进程发送信号
>
> 如果进程处于可中断睡眠状态，则设置信号位后唤醒进程
>
> 如果进程处于不可中断睡眠状态，则只设置信号位
>
> 进程处理信号的时机在其从内核态返回用户态时
>
> * 收到信号后进程退出
> * 进程忽略信号
> * 捕捉某类信号 - 调用对应的信号处理函数
>
> 一种可能的思路：
>
> 向进程发送某个信号后，会将该信号的阻塞位置位
>
> 进程处理完该信号后，将该阻塞位复位
>
> 如果进程处理信号期间，又来了同样的信号
>
> 那么会被阻塞位给屏蔽掉，即丢弃
>
> (所谓的不可靠信号)
>
> 可靠的信号处理形式：排队记录，那么就不存在丢弃的问题

OK，看代码，一开始是内核调试函数

用于显示各任务的详细信息：

```c
void show_task(int nr, struct task_struct * p)
{
    int i, j = 4096-sizeof(struct task_struct); // 内核栈最大容量
                                                // PCB 和内核态堆栈共占一页物理内存
    
    printk("%d: pid=%d, state=%d, father=%d, child=%d, ",
            nr, p->pid, p->state, p->p_pptr->pid,
            p->p_cptr ? p->p_cptr->pid : -1);
    
    i = 0;
    while (i < j && !((char *)(p+1))[i])
        i++; // 检测指定任务数据结构以后等于 0 的字节数 (大约)
             // 即，内核态堆栈空闲字节数
    
    printk("%d/%d chars free in kstack\n\r", i, j);
    printk(" PC=%08X.", *(1019 + (unsigned long *) p));
    if (p->p_ysptr || p->p_osptr)
        printk(" Younger sib=%d, older sib=%d\n\r",
                p->p_ysptr ? p->p_ysptr->pid : -1,
                p->p_osptr ? p->p_osptr->pid : -1);
    else
        printk("\n\r");
}

void show_state(void)
{
    int i;
    
    printk("\rTask-info:\n\r");
    for (i = 0; i < NR_TASKS; i++) // NR_TASKS 为内核支持的最多任务数
        if (task[i]) // 任务指针不为空 (任务存在)
            show_task(i, task[i]);
}
```

接下来是一些相关数据结构的定义：

```c
union task_union {
    struct task_struct task;
    char stack[PAGE_SIZE];
};
// ???
// union 大一上 C 语言课的时候一笔带过
// 有空再弄明白这具体是个啥

// 任务 0 初始化
static union task_union init_task = { INIT_TASK, };

// 初始化系统滴答
// volatile 和编译器有关，需要有空弄弄明白
// 系统滴答 10ms 一次，是系统时钟单位
unsigned long volatile jiffies = 0;
// 开机时间
unsigned long startup_time = 0;
// 为调整时钟而需要增加的滴答数
int jiffies_offset = 0;

// 当前任务指针，指向任务 0
struct task_struct *current = &(init_task.task);
// 上一个使用协处理器的任务指针
struct task_struct *last_task_used_math = NULL; 
// 定义任务指针数组，第一项被初始化为任务 0
struct task_struct *task[NR_TASKS] = { &(init_task.task), };

// 任务 0 的用户态堆栈
// 1K 项，共 4KB
long user_stack[ PAGE_SIZE >> 2 ];

// SS:ESP
// SS 的段选择符为内核数据段选择符 0x10
// ESP 是逆向入栈的 (地址递减)，因此初始地址指向 user_stack 的最后一项
struct {
    long *a;
    short b;
} stack_start = { & user_stack [ PAGE_SIZE >> 2 ], 0x10 };
```

接下来是一个子函数

用于在任务被调度切换之后

保存原任务的协处理器状态，恢复新任务的协处理器状态

```c
void math_state_restore()
{
    if (last_task_used_math == current)
    {
        return; // 任务没变，直接返回
    }
    __asm__("fwait"); // 发送协处理器指令之前要先发 WAIT 指令
    if (last_task_used_math)
    {
        __asm__("fnsave %0"::"m" (last_task_used_math->tss.i387));
    }
    last_task_used_math = current; // 指向当前任务
    if (current->used_math) { // 当前任务使用过协处理器
        __asm__("frstor %0"::"m" (current->tss.i387)); // 恢复协处理器状态
    } else { // 首次使用协处理器
        __asm__("fninit"::); // 初始化协处理器
        current->used_math = 1; // 设置已使用协处理器标志
    }
}
```

下面是最核心的 `schedule()` 函数 (会被很多地方调用到！)：

* 调度进程
* 处理信号

```c
void schedule(void)
{
    int i, next, c;
    struct task_struct **p;
    
    // 首先检测各进程的定时器
    // 唤醒已得到信号的可中断任务
    for (p = &LAST_TASK; p > &FIRST_TASK; --p)
        if (*p) {
            if ((*p)->timeout && (*p)->timeout < jiffies) {
                // 已设置定时器 && 已经超时
                (*p)->timeout = 0; // 复位定时器
                if ((*p)->state == TASK_INTERRUPTIBLE)
                    (*p)->state = TASK_RUNNING;
            }
            if ((*p)->alarm && (*p)->alarm < jiffies) {
                // 已设置 SIGALRM 信号 && 已经超时
                (*p)->signal |= (1 << (SIGALRM-1)); // 置 SIGALRM 信号
                (*p)->alarm = 0;
            }
            if (((*p)->signal & ~(_BLOCKALBE & (*p)->blocked)) &&
                (*p)->state == TASK_INTERRUPTIBLE})
                // 信号 bitmap 中存在除被阻塞的信号外还有信号存在
                // 且任务处于可中断等待状态
                (*p)->state = TASK_RUNNING;
        }
    
    // 调度程序的主要部分
    while (1) {
        c = -1; // 数值最大的时间片
        next = 0; // 下一个要调度的任务
        i = NR_TASKS; // 任务 index
        p = &task[NR_TASKS]; // 最后一个任务项
        
        // 从最后一个任务开始循环
        while (--i) {
            if (!&--p)
                continue; // 跳过空槽
            if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
                c = (*p)->counter, next = i;
        }
        
        // 有大于 0 的时间片，则跳出循环，将任务切换到 next
        // 或没有一个可运行任务 (c == -1, next == 0)，则跳出循环，切换到任务 0
        if (c) break;

        // counter == 0，即所有进程的时间片都为 0
        // 根据优先级重新计算 counter
        // counter = counter/2 + priority
        for (p = &LAST_TASK; p > &FIRST_TASK; --p)
            if (*p)
                (*p)->counter = ((*p)->counter >> 1) + (*p)->priority;
    }
    
    switch_to(next); // 任务切换宏
}
```

