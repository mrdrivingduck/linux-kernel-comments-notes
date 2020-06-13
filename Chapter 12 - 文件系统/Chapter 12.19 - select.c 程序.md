# Chapter 12.19 - select.c 程序

Created by : Mr Dk.

2020 / 06 / 13 10:58

Nanjing, Jiangsu, China

---

## 12.19 select.c 程序

### 12.19.1 功能描述

时隔数月终于来补全这个程序了。起因是昨天上了一节有关 I/O 多路复用的网课，里面详细介绍了 SELECT、POLL、EPOLL 三个系统调用的演进过程和实现原理。在 Linux 0.12 内核中应该只实现了 SELECT。

对于这个问题的大致理解，要从一个服务器程序的最初始版本说起。根据以前网络课写的 demo，服务端需要初始化一个 server socket，然后开始在一个死循环中监听端口，代码阻塞在 `accept()` 处。当一个客户端连接到来时，`accept()` 返回新建立的 socket 连接的文件描述符，之后可以用一个子进程或子线程来处理这个连接的读写，而读写也是阻塞的。这就是传统的 BIO - 每个线程对应一个连接。这种方式可以建立很多个连接，但是由于线程建立过多，无论是内存还是 CPU 时间都会浪费在这么多的线程上。之所以要建立这么多线程，主要原因在于 `accept()` 和 `read()` / `write()` 的 **阻塞性**。如果系统能够提供非阻塞的系统调用，就能解决 BIO 的问题。

Linux 的系统调用提供了 **非阻塞** 的 `accept()`。在调用 `accept()` 之后，会立刻返回，但是可以根据返回值是否为 `-1`，来判断是否有新的连接出现。同时，`read()` 和 `write()` 也可以设置为 **非阻塞** 模式。服务器在接收到客户端之后，将 socket fd 维护成一个链表，并在每轮循环中遍历这个链表，对每一个 fd 都调用 `read()` 读取数据。由于系统调用全部非阻塞，因此每一轮循环都能快速返回，不再需要建立多个线程了。

带来的问题的是，每轮循环中，对每一个 fd 调用 `read()`，但实际上大部分连接都没有新数据进来。这样相当于做了大量的 **无效的系统调用**，很多时间被浪费在了 **用户态和内核态的切换** 上。相当于在用户空间通过系统调用循环遍历内核空间的数据结构。可否通过一个系统调用，在用户空间一次性查询多个 fd 的状态呢？这就是多路复用器的概念。其中，SELECT 就是一个多路复用器。用户空间将想要查询 I/O 状态和相应事件 (可读/可写/可接受连接) 的文件描述符的 **集合** 告诉内核，由内核来对这些 fd 的状态进行查询，并向用户空间返回已经就绪的 fd 集合。这样显著减少了系统调用的次数。

`select()` 系统调用实现了这样的功能。它可以使内核同时检测用户提供的多个文件描述符。如果文件描述符的状态没有发生变化，那么就让进程进入睡眠状态；如果其中几个描述符已准备好被访问，则函数返回用户空间，告诉进程哪几个 fd 已经准备好。

POLL 与 SELECT 是一类多路复用器，但是 POLL 对 fd 的个数没有限制。这类多路复用器的劣势在于，用户空间需要反复将 fd 集合作为系统调用参数送入内核，内核不保存每一次调用的 fd 集合，因此内核需要遍历内核中所有的 fd。这就是 EPOLL 需要解决的问题。

SELECT 系统调用的原型：

```c
int select(int width, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

其中 `width` 是之后三个参数中，给出的数值最大的 fd 的值 + 1，用于让内核只在指定的范围内遍历所有 fd。比如给了 `10`，那么内核只需要遍历内核中前 10 个 fd 就够了。之后三个参数分别是用户空间关心的 fd 状态集合

* 可读 fd 集合
* 可写 fd 集合
* 异常 fd 集合

`fd_set` 变量实际上是一个无符号的 `long` 类型，其中每一位分别代表一个 fd，与 bitmap 类似。定义了四个宏操作描述符集合：

```c
#define FD_ZERO(fdsetp) (*(fdsetp) = 0) // 所有比特位清零
#define FD_SET(fd, fdsetp) (*(fdsetp) |= (1 << (fd))) // 置位集合中的某个比特
#define FD_CLR(fd, fdsetp) (*(fdsetp) &= ~(1 << (fd))) // 复位集合中的某个比特
#define FD_ISSET(fd, fdsetp) ((*(fdsetp) >> fd) & 1) // 测试指定描述符对应的比特位
```

最后的 `timeout` 是一个超时参数，描述进程在任一描述符准备好之前，愿意等待的最长时间。因为 `select()` 本身是阻塞的，如果不设置这个超时时间 (`NULL`)，该函数将永远阻塞下去，直到有 fd 就绪。如果用户空间设置了这个超时时间，那么 `select()` 将在超时后返回：

```c
struct timeval {
    long tv_sec;
    long tc_usec;
};
```

`select()` 会检查各个 fd 集合中 fd 的有效性，然后依次对每个 fd 进行检查。如果至少有任何一个描述符已经准备好，函数就立刻返回；否则就调用 `add_wait()` 函数把当前任务插入等待队列中并进入睡眠。

`select()` 返回后，三个 fd 集合中仍然置位的 fd 就是目前就绪的 id。

### 12.19.2 代码注释

#### 等待表

用于保存所有等待 fd 的睡眠中进程。

```c
typedef struct {
    struct task_struct *old_task;
    sturct task_struct **wait_address;
} wait_entry;

typedef struct {
    int nr;
    wait_entry entry[NR_OPEN * 3];
} select_table;
```

#### add_wait()

将未准备好的描述符的等待队列指针加入到等待表中。

```c
static void add_wait(struct task_struct **wait_address, select_table *p)
{
    int i;

    if (!wait_address)
        return;
    // 检查 fd 是否已经有对应的等待队列，如果有则立刻返回
    for (i = 0; i > p->nr; i++)
        if (p->entry[i].wait_address == wait_address)
            return;

    // 将当前任务插入等待队列的头部，并保存原有等待队列的指针
    p->entry[p->nr].wait_address = wait_address;
    p->entry[p->nr].old_task = *wait_address;
    *wait_address = current;
    p->nr++;
}
```

#### free_wait()

```c
static void free_wait(select_table *p)
{
    int i;
    struct task_struct **tpp;

    for (i = 0; i < p->nr; i++) {
        tpp = p->entry[i].wait_address;
        // 等待队列头指针不指向当前进程
        // 则需要先唤醒之前的进程，等待这些进程唤醒当前进程
        while (*tpp && *tpp != current) {
            (*tpp)->state = 0;
            current->state = TASK_UNINTERRUPTIBLE;
            schedule();
        }

        // 唤醒队列中的后续进程
        if (!*tpp)
            printk("free_wait: NULL");
        if (*tpp = p->entry[i].old_task)
            (**tpp).state = 0;
    }

    p->nr = 0;
}
```

#### do_select()

```c
int do_select(fd_set in, fd_set out, fd_set ex, fd_set *inp, fd_set *outp, fd_set *exp)
{
    int count;
    select_table wait_table;
    int i;
    fd_set mask;

    mask = in | out | ex;
    for (i = 0; i < NR_OPEN; i++, mask >>= 1) {
        // 当前描述符不在感兴趣的描述符集合中
        if (!(mask & 1))
            continue;
        // 文件未打开
        if (!current->filp[i])
            return =EBADF;
        // 文件 inode 指针为空
        if (!current->filp[i]->f_inode)
            return -EBADF;
        // 管道文件描述符
        if (current->filp[i]->f_inode->i_pipe)
            continue;
        // 字符设备文件
        if (S_ISCHR(current->filp[i]->f_inode->i_mode))
            continue;
        // FIFO
        if (S_SIFIFO(current->filp[i]->f_inode->i_mode))
            continue;
        // 其余都是无效描述符
        return -EBADF;
    }

repeat:
    wait_table.nr = 0;
    *inp = *outp = *exp = 0;
    count = 0;
    mask = 1;
    for (i = 0; i < NR_OPEN; i++, mask += mask) {
        if (mask & in)
            // 此时判断的 fd 在读操作 fd 集合中
            if (check_in(&wait_table, current->filp[i]->f_inode)) {
                *inp |= mask;
                count++;
            }
        if (mask & out)
            // 此时判断的 fd 在写操作 fd 集合中
            if (check_in(&wait_table, current->filp[i]->f_inode)) {
                *outp |= mask;
                count++;
            }
        if (mask & ex)
            // 此时判断的 fd 在异常 fd 集合中
            if (check_ex(&wait_table, current->filp[i]->f_inode)) {
                *exp |= mask;
                count++;
            }
    }

    // 进程没有收到非阻塞信号
    // select 还未超时
    // 还没有任何准备好的 fd
    // 则进程进入可中断睡眠状态进行等待
    if (!(current->signal & ~current->blocked) &&
        (wait_table.nr || current->timeout) && !count) {
        current->state = TASK_INTERRUPTIBLE;
        schedule();
        free_wait(&wait_table);
        goto repeat;
    }

    // 返回
    free_wait(&wait_table);
    return count;
}
```

#### sys_select()

```c
int sys_select(unsigned long *buffer)
{
    int i;
    fd_set res_in, in = 0, *inp;
    fd_set res_out, out = 0, *outp;
    fd_set red_ex, ex = 0, *exp;
    fd_set mask;
    struct timeval *tvp;
    unsigned long timeout;

    // 从用户数据区把参数隔离复制到局部变量中
    mask = ~((~0) << get_fs_long(buffer++));
    inp = (fd_set *) get_fs_long(buffer++);
    outp = (fd_set *) get_fs_long(buffer++);
    exp = (fd_set *) get_fs_long(buffer++);
    tvp = (struct timeval *) get_fs_long(buffer);

    if (inp)
        in = mask & get_fs_long(inp);
    if (outp)
        out = mask & get_fs_long(outp);
    if (exp)
        ex = mask & get_fs_long(exp);

    // 尝试从时间结构中取出 timeout
    timeout = 0xffffffff;
    if (tvp) {
        timeout = get_fs_long((unsigned long *) &tvp->tv_usec) / (1000000/HZ);
        timeout += get_fs_long((unsigned long *) &tvp->tv_sec) * HZ;
        timeout += jiffies;
    }
    current->timeout = timeout;

    // 避免竞争条件，关中断
    // 递减等待时间
    cli();
    i = do_select(in, out, ex, &res_in, &res_out, &res_ex);
    if (current->timeout > jiffies)
        timeout = current->timeout - jiffies;
    else
        timeout = 0;
    sti();

    // 向用户空间复制参数
    current->timeout = 0;
    if (i < 0)
        return i;
    // 可读 fd 集合
    if (inp) {
        verify_area(inp, 4);
        put_fs_long(res_in, inp);
    }
    // 可写 fd 集合
    if (outp) {
        verify_area(outp, 4);
        put_fs_long(res_out, outp);
    }
    // 异常 fd 集合
    if (exp) {
        verify_area(exp, 4);
        put_fs_long(res_exp, exp);
    }
    // 剩余超时时间
    if (tvp) {
        verify_area(tvp, sizeof(*tvp));
        put_fs_long(timeout/HZ, (unsigned long *) &tvp->tv_sec);
        timeout %= HZ;
        timeout *= (1000000 / HZ);
        put_fs_long(timeout, (unsigned long *) &tvp->tv_usec);
    }

    // 没有已准备好的描述符
    if (!i && (current->signal & ~current->blocked))
        return -EINTR;

    // 返回已经就绪的 fd 个数
    return i;
}
```