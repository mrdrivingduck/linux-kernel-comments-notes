# Chapter 9.4 - ll_rw_blk.c 程序

Created by : Mr Dk.

2019 / 08 / 23 20:22

Ningbo, Zhejiang, China

---

## 9.4 ll_rw_blk.c 程序

### 9.4.1 功能描述

用于执行低层块设备读/写操作

是所有块设备与系统其它部分的接口程序

通过调用 `ll_rw_block()` 函数

系统中其它程序可以异步读写块设备中的数据

主要功能：

* 创建读写请求项
* 插入到指定设备的请求队列中

实际读写操作由对应设备的请求项处理函数 `do_XX_request()` 完成

`ll_rw_block()` 函数：

* 针对一个块设备，建立起一个请求项
* 测试块设备的当前请求项
  * 若为空，则设备空闲，将请求项作为设备的当前请求项，并直接调用请求项处理函数
  * 若不为空，则使用电梯算法将请求项插入请求项链表中，等待处理

请求项处理函数结束后

* 在中断处理过程中，通过回调 C 函数再次调用请求项处理函数，处理链表中的其余项
* 只要请求项列表不为空，都会被陆续处理

当请求项链表为空时，请求项处理程序不再向设备控制器发送命令，而是立刻退出

### 9.4.2 代码注释

#### 几个数据结构的定义

```c
// 32 个设备请求项
struct request request[NR_REQUEST];

// 请求数组没有空闲项时的临时等待处
struct task_struct * wait_for_request = NULL; 

// 块设备数组，每项中包含 设备处理函数指针 和 设备当前请求项指针
struct blk_dev_struct blk_dev[NR_BLK_DEV] = {
    { NULL, NULL }, // 0 - 无设备
    { NULL, NULL }, // 1 - 内存
    { NULL, NULL }, // 2 - 软驱
    { NULL, NULL }, // 3 - 硬盘
    { NULL, NULL }, // 4 - ttyx 设备
    { NULL, NULL }, // 5 - tty 设备
    { NULL, NULL }  // 6 - lp 打印机设备
};

// 所有块设备的逻辑块总数 (1 块 = 1 KB)
// blk_size[MAJOR][MINOR] 为某个子设备上的数据块数
int * blk_size[NR_BLK_DEV] = { NULL, NULL, };
```

#### 缓冲块的加锁与解锁函数 lock_buffer() & unlock_buffer()

```c
static inline void lock_buffer(struct buffer_head * bh)
{
    cli(); // 关中断
    while (bh->b_lock)
        // 缓冲区已被锁定，则睡眠，直到缓冲区解锁
        // 睡眠结束后仍在循环中
        sleep_on(&bh->b_wait);
    bh->b_lock = 1; // 锁定缓冲区
    sti(); // 开中断
}
```

```c
static inline void unlock_buffer(struct buffer_head * bh)
{
    if (!bh->b_lock)
        printk("ll_rw_block.c: buffer not locked\n\r");
    bh->b_lock = 0; // 解锁
    wake_up(&bh->b_wait); // 唤醒等待缓冲区的任务
}
```

#### _附 - 电梯算法排序函数宏 IN_ORDER_

参数分别为两个请求项结构体的指针

用于在请求项链表中判断顺序

注意优先级 : `&&` ＞ `||`

显然，按照逻辑，首先判断请求的命令

* `READ(0)` 优先于 `WRITE(1)`

如果命令相同，则再判断设备号

* 比如，应当按顺序依次操作各个分区？

如果设备号也相同，再比较扇区号

* 应当按顺序操作分区

这样调度，磁头移动的总距离比较小

>  个人理解：可以把这个宏理解为 `<` 运算符的重载

```c
#define IN_ORDER(s1, s2) ( \
    (s1)->cmd < (s2)->cmd || \
    (s1)->cmd == (s2)->cmd && ( \
        (s1)->dev < (s2)->dev || \
        (s1)->dev == (s2)->dev && (s1)->sector < (s2)->sector \
    ) \
)
```

#### 将请求项插入链表子函数 add_request()

参数给出指定的 __块设备结构__ 指针和已经设置好的 __请求项结构__ 指针

如果设备的 __当前请求项__ 为空，则设置当前请求项后，立刻调用请求处理函数

否则将请求项插入链表中

```c
static void add_request(struct blk_dev_struct * dev, struct request * req)
{
    struct request * tmp;
    
    req->next = NULL; // 防止请求项被销毁时指针错乱
    cli(); // 关中断
    if (req->bh)
        req->bh->b_dirt = 0;
    if (!(tmp = dev->current_request)) {
        // 设备当前请求项为空
        dev->current_request = req; // 设备当前请求项指向当前请求项
        sti(); // 开中断
        (dev->request_fn)(); // 立刻调用设备请求处理函数
        return;
    }
    
    // 设备当前请求项不为空
    // tmp 目前指向链表头，即当前正被设备处理的请求
    for ( ; tmp->next; tmp = tmp->next) {
        // tmp->next 为每次进行判断的项
        if (!req->bh)
            // 请求项的缓冲块头指针为空
            // 需要找一个项，已经有可用的缓冲块
            // 意思是，没有缓冲块的请求项最优先？？？
            if (tmp->next->bh)
                break;
            else
                continue;
        // 电梯算法，IN_ORDER 相当于 < 运算符
        // (tmp < req || tmp >= tmp->next) && req < tmp->next
        if ((IN_ORDER(tmp, req) || !IN_ORDER(tmp, tmp->next)) &&
            IN_ORDER(req, tmp->next))
            break;
    }
    // 将 req 插入到 tmp 和 tmp->next 之间
    req->next = tmp->next;
    tmp->next = req;
    
    sti(); // 开中断
}
```

#### 创建请求项并插入请求队列 make_request()

参数为：

* 主设备号
* 命令
* 存放数据的缓冲区头指针

创建请求项时，为保证 __读请求__ 的优先性

请求队列的后 1/3 仅用于读请求

在请求数组中搜索空闲项时

* 对于读请求，从数组尾部开始搜索
* 对于写请求，从数组的 2/3 处开始搜索

这样能保证请求队列的后 1/3 全部是读请求

```c
static void make_request(int major, int rw, struct buffer_head * bh)
{
    // 判断命令合法性
    if (rw_ahead = (rw == READA || rw == WRITEA)) {
        if (bh->b_lock)
            return;
        if (rw == READA)
            rw = READ;
        else
            rw = WRITE;
    }
    if (rw != READ && rw != WRITE)
        panic("Bad block dev command, must be R/W/RA/WA");
    
    lock_buffer(bh);
    
    if ((rw == WRITE && !bh->b_dirt) || (rw == READ && bh->b_uptodate)) {
        unlock_buffer(bh);
        return;
    }
    
repeat:
    // 搜索空闲请求项
    if (rw == READ)
        req = request + NR_REQUEST; // 数组尾部
    else
        req = request + ((NR_REQUEST * 2) / 3); // 数组 2/3 处
    while (--req >= request)
        if (req->dev < 0)
            // 找到了空闲项
            break;
    if (req < request) {
        // 搜索已经超出了数组首地址，说明没有空闲项
        if (rw_ahead) {
            unlock_buffer(bh);
            return;
        }
        // 使该进程睡眠，并加入请求队列中
        sleep_on(&wait_for_request);
        // 进程被唤醒后，从这里继续执行
        // 重新搜索空闲请求项
        goto repeat;
    }
    
    // req 已经指向找到的空闲请求项
    // 向空闲请求项中填写请求信息，并加入队列
    req->dev = bh->b_dev; // 设备号
    req->cmd = rw; // 命令
    req->errors = 0; // 操作产生的错误次数
    req->sector = bh->b_blocknr << 1; // 起始扇区 (缓冲块转换为扇区)
    req->nr_sectors = 2; // 读写一个数据块 (2 个扇区)
    req->buffer = bh->b_data; // 需要读写的数据缓冲区
    req->waiting = NULL; // 等待操作完成的任务
    req->bh = bh; // 缓冲块头指针
    req->next = NULL; // 下一请求项
    add_request(blk_dev + major. req); // blk_dev[major]
}
```

#### 低级数据块读写函数 ll_rw_block()

每次读写 1KB 的数据块 (2 个扇区)

是块设备驱动程序与系统其它部分之间的接口函数

创建请求项，并插入指定块设备的请求项链表中

在调用该函数之前，调用者需要把读/写块设备的信息保存在缓冲区头结构中

```c
void ll_rw_block(int rw, struct buffer_head * bh)
{
    unsigned int major;
    
    if ((major = MAJOR(bh->b_dev)) >= NR_BLK_DEV || !(blk_dev[major].request_fn)) {
        // 主设备号不存在 || 该设备的请求处理函数不存在
        printk("Trying to read nonexistent block-device\n\r");
        return;
    }
    
    make_request(major, rw, bh);
}
```

#### 低级页面读写函数 ll_rw_page()

每次读写 4KB 的页 (8 个扇区)

```c
void ll_rw_page(int rw, int dev, int page, char * buffer)
{
    struct request * req;
    unsigned int major = MAJOR(dev);
    
    // 参数合法性检测
    if (major >= NR_BLK_DEV || !(blk_dev[major].request_fn)) {
        // 主设备号不存在 || 设备的请求操作函数不存在
        printk("Trying to read nonexistent block-device\n\r");
        return;
    }
    if (rw != READ && rw != WRITE)
        // 命令不是 READ 也不是 WRITE，内核出错停机
        panic("Bad block dev command, must be R/W");
    
repeat:
    // 搜索空闲请求项
    req = request + NR_REQUEST; // 指向请求数组结尾
    while (--req >= request)
        if (req->dev < 0)
            // 空闲项
            break;
    if (req < request) {
        // 未找到空闲项
        // 睡眠，并加入等待队列
        sleep_on(&wait_for_request);
        // 被唤醒后从该处继续执行，重新搜索空闲请求项
        goto repeat;
    }
    
    // 找到空闲请求项
    // 填写信息并加入请求项链表中
    req->dev = dev;
    req->cmd = rw;
    req->errors = 0;
    req->sector = page << 3;
    req->nr_sectors = 8;
    req->buffer = buffer;
    req->waiting = current;
    req->bh = NULL;
    req->next = NULL;
    current->state = TASK_UNINTERRUPTIBLE; // 当前进程置为不可中断等待
    add_request(blk_dev + major, req); // 加入请求项链表
    schedule(); // 调度其它进程运行
}
```

> 这个函数就很有意思了
>
> 和上面的读写数据块函数有什么区别呢？
>
> 首先，请求项的 `waiting` 字段，页面读写函数加入了当前进程
>
> 而且在之后，还改变了进程的状态，并调用了进程调度函数
>
> 所以说，不知道可不可以这样理解：
>
> * `ll_rw_block()` 的调用者是内核
> * `ll_rw_page()` 的调用者是进程
>
> ？

#### 块设备初始化函数 blk_dev_init()

由初始化程序 `main.c` 调用

主要功能是初始化请求项数组

* 将所有请求项置为空闲 (dev == -1)
* 将所有链表指针置为 NULL

```c
void blk_dev_init(void)
{
    int i;
    
    for (i = 0; i < NR_REQUEST; i++) {
        request[i].dev = -1;
        request[i].next = NULL;
    }
}
```

---

