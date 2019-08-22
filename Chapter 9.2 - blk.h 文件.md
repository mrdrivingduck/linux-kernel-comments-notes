# Chapter 9.2 - blk.h 文件

Created by : Mr Dk.

2019 / 08 / 22 14:49

Ningbo, Zhejiang, China

---

## 9.2 blk.h 文件

### 9.2.1 功能描述

块设备参数的头文件

定义了请求等待队列中的请求项数据结构 request

用宏语句定义了电梯搜索算法

对内核目前支持的设备，根据其各自的主设备号分别定义了对应的常数值

### 9.2.2 代码注释

#### 块设备表

每种块设备占用一项，index 即为主设备号

```c
#define NR_BLK_DEV 7

struct blk_dev_struct {
    void (*reuqest_fn) (void); // 设备的请求处理函数
    struct request * current_request; // 设备当前正在处理的请求
};

extern struct blk_dev_struct blk_dev[NR_BLK_DEV];
```

#### 设备请求项结构体及其数组

```c
#define NR_REQUEST 32

struct request {
    int dev; // 发出请求的设备号，-1 代表空闲
    int cmd; // READ | WRITE
    int errors; // 操作产生的错误次数
    unsigned long sector; // 起始扇区
    unsigned long nr_sectors; // 操作扇区数
    char * buffer; // 数据缓冲区
    struct task_struct * waiting; // 等待请求完成的任务
    struct buffer_head * bh; // 缓冲区头指针
    struct request * next; // 指向下一个请求项
};

// 请求项数组
extern struct request request[NR_REQUEST];

// 等待空闲请求项的进程队列头指针
// 32 个请求项全被占满时，进程在队列中排队
extern struct task_struct * wait_for_request;
```

#### 数据块总数指针数组

指向对应主设备号的总块数数组 hd_sizes[]

```c
extern int * blk_size[NR_BLK_DEV];
```

#### 设备宏

```c
#ifdef MAJOR_NR

#if (MAJOR_NR == 1)
#define DEVICE_NAME "ramdisk"
#define DEVICE_REQUEST do_rd_request // 虚拟盘请求项处理函数
#define DEVICE_NR(device) ((device) & 7) // 子设备号 0 - 7
#define DEVICE_ON(device) // 虚拟盘无需开启和关闭
#define DEVICE_OFF(device)

#elif (MAJOR_NR == 2)
#define DEVICE_NAME "floppy"
#define DEVICE_INTR do_floppy // 软盘中断处理函数
#define DEVICE_REQUEST do_fd_request // 软盘请求项处理函数
#define DEVICE_NR(device) ((device) & 3) // 子设备号 0 - 3
#define DEVICE_ON(device) floppy_on(DEVICE_NR(device)) // 开启设备宏
#define DEVICE_OFF(device) floppy_off(DEVICE_NR(device)) // 关闭设备宏

#elif (MAJOR_NR == 3)
#define DEVICE_NAME "harddisk"
#define DEVICE_INTR do_hd // 硬盘中断处理函数
#define DEVICE_TIMEOUT hd_timeout // 硬盘超时值
#define DEVICE_REQUEST do_hd_request // 硬盘请求项处理函数
#define DEVICE_NR(device) (MINOR(device) / 5) // 设备号 0 - 1
#define DEVICE_ON(device) // 硬盘一开机总是运转着
#define DEVICE_OFF(device)

#elif
#error "unknown blk device"

#endif
```

#### 当前请求项的宏

```c
#define CURRENT (blk_dev[MAJOR_NR].current_request) // 主设备的当前请求指针
#define CURRENT_DEV DEVICE_NR(CURRENT->dev) // 当前请求项的设备号
```

#### 结束请求处理函数 end_request()

首先是一个解锁缓冲块的子函数

> 这个缓冲块具体指的是什么还没太明白
>
> 请求结构里又有一个数据缓冲块的指针，又有一个缓冲区头指针
>
> 不知道有什么区别

```c
extern inline void unlock_buffer(struct buffer_head * bh)
{
    if (!bh->b_lock)
        printk(DEVICE_NAME ": free buffer being unlocked\n");
    bh->b_block = 0;
    wake_up(&bh->b_wait);
}
```

* 首先关闭指定的块设备
* 检查此次读写缓冲区是否有效
  * 有效则设置缓冲区更新标志，解锁该缓冲区
  * 否则显示相关的 I/O 出错信息
* 唤醒等待该请求项的进程
* 唤醒等待空闲请求项的进程
* 释放并从链表中清除本请求项
* 处理下一个请求项

```c
extern inline void end_request(int uptodate)
{
    DEVICE_OFF(CURRENT->dev); // 关闭设备
    if (CURRENT->bh) {
        CURRENT->bh->b_uptodate = uptodate; // 设置缓冲区更新标志
        unlock_buffer(CURRENT->bh); // 解锁缓冲区
    }
    if (!uptodate) {
        // 出错
        printk(DEVICE_NAME "I/O error\n\r");
        printk("dev %04x, block %d\n\r", CURRENT->dev, CURRENT->bh->b_blocknr);
    }
    wake_up(&CURRENT->waiting); // 唤醒等待该请求项的进程
    wake_up(&wait_for_request); // 唤醒等待空闲请求项的进程
    CURRENT->dev = -1; // 释放该请求项
    CURRENT = CURRENT->next; // 指向下一个请求项
}
```

#### 请求项初始化宏

几个块设备驱动程序的开始处

对请求项的初始化操作 __类似__

因此定义一个统一的初始化宏，并对当前请求项进行一些有效性判断

* 若当前请求项为空，则设备已经没有需要处理的请求项了，扫尾后退出函数
* 主设备号 ≠ 驱动程序定义的主设备号，说明请求项队列错乱，内核 panic
* 若请求项中使用的缓冲块没有被锁定，说明 kernel 也有问题，panic

```c
#define INIT_REQUEST \
repeat: \
    if (!CURRENT) { \ // 当前请求项为空
        CLEAR_DEVICE_INTR \
        CLEAR_DEVICE_TIMEOUT \
        return; \
    } \
    if (MAJOR(CURRENT->dev) != MAJOR_NR)\ // 主设备号不对
        panic(DEVICE_NAME ": request list destroyed"); \
    if (CURRENT->bh) { \
        if (!CURRENT->bh->b_lock) \ // 缓冲区未锁定
            panic(DEVICE_NAME ": block not locked"); \
    }
#endif
```

---

