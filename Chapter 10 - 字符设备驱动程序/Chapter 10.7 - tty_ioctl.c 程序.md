# Chapter 10.7 - tty_ioctl.c 程序

Created by : Mr Dk.

2019 / 08 / 27 14:05

Ningbo, Zhejiang, China

---

## 10.7 tty_ioctl.c 程序

### 10.7.1 功能描述

实现了函数 `tty_ioctl()`，用于字符设备的控制操作

程序可以通过该函数修改指定终端 termios 结构体中的设置标志

该函数将由输入输出控制系统调用 `sys_ioctl()` 调用

用于实现基于 __文件系统__ 的统一设备访问接口

一般用户程序不会直接使用 `sys_ioctl()`

* 使用库函数中的封装函数
* 使用 `ioctl()` 库函数

### 10.7.2 代码注释

#### 修改传输波特率 change_speed()

首先定义了串行端口使用的 __波特率因子数组__

```c
static unsigned short quotient[] = {
    0, 2304, 1536, 1047, 857,
    768, 576, 384, 192, 96,
    64, 48, 24, 12, 6, 3
};
```

在除数锁存标志 DLAB 置位的情况下

对串口的两个端口分别写入波特率因子的低字节和高字节

写完后复位 DLAB 位

```c
static void change_speed(struct tty_struct * tty)
{
    unsigned short port, quot;
    
    if (!(port = tty->read_q->data))
        // 不是串行终端，退出
        // 串行终端的 tty 结构读队列的 data 字段存放着串行端口的基址
        // 一般控制台终端的该字段为 0
        return;
    
    quot = quotient[tty->termios.c_cflag & CBAUD];
    
    cli();
    outb_p(0x80, port + 3); // DLAB 置位
    outb_p(quot & 0xff, port); // 波特率因子低字节
    outb_p(quot >> 8, port + 1); // 波特率因子高字节
    outb(0x03, port + 3); // DLAB 复位
    sti();
}
```

#### 刷新 tty 缓冲队列 flush()

令缓冲队列的头指针等于尾指针，从而清空缓冲区

```c
static void flush(struct tty_queue * queue)
{
    cli();
    queue->head = queue->tail;
    sti();
}
```

#### 读取/设置终端 termios 结构信息 get_termios() / set_termios()

```c
static int get_termios(struct tty_struct * tty, struct termios * termios)
{
    int i;
    
    verify_area(termios, sizeof(*termios));
    for (i = 0; i < (sizeof(*termios)); i++)
        put_fs_byte( ((char *)&tty->termios)[i], i+(char *)termios );
    return 0;
}
```

```c
static int set_termios(struct tty_struct * tty, struct termios * termios, int channel)
{
    int i, retsig;
    
    if ((current->tty == channel) && (tty->pgrp != current->pgrp)) {
        // 试图设置终端状态，但终端不在前台
        // 发送 SIGTTOU 信号
        retsig = tty_signal(SIGTTOU, tty);
        if (retsig == -ERESTARTSYS || retig == -EINTR)
            return retsig;
    }
    
    for (i = 0; i < (sizeof (*termios)); i++)
        ((char *) &tty->termios)[i] = get_fs_byte(i+(char *)termios);
    change_speed(tty); // 修改波特率
    return 0;
}
```

#### 读取/设置终端 termio 结构信息 get_termio() / set_termios()

termio 结构与 termios 基本相同

但标志集的数据类型不同

所以要经过类型转换

```c
static int get_termio(struct tty_struct * tty, struct termio * termio)
{
    int i;
    struct termio tmp_termio;
    
    verify_area(termio, sizeof(*termio));
    tmp_termio.c_iflag = tty->termios.c_iflag;
    tmp_termio.c_oflag = tty->termios.c_oflag;
    tmp_termio.c_cflag = tty->termios.c_cflag;
    tmp_termio.c_lflag = tty->termios.c_lflag;
    tmp_termio.c_line = tty->termios.c_line;
    
    for (i = 0; i < NCC; i++)
        tmp_termio.c_cc[i] = tty->termios.c_cc[i];
    for (i = 0; i < sizeof(*termio); i++)
        put_fs_byte(((char *)&tmp_termio)[i], i+(char *)termio);
    return 0;
}
```

```c
static int set_termio(struct tty_struct * tty, struct termio * termio, int channel)
{
    int i, retsig;
    struct termio tmp_termio;
    
    if ((current->tty == channel) && (tty->pgrp != current->pgrp)) {
        // 当前进程不在前台
        // 发送 SIGTTOU 信号让使用终端的进程先暂时停止执行
        retsig = tty_signal(SIGTTOU, tty);
        if (retsig == -ERESTARTSYS || retsig == -EINTR)
            return retsig;
    }

    for (i = 0; i < (sizeof (*termio)); i++)
        ((char *)&tmp_termio)[i]=get_fs_byte(i+(char *)termio);
    
    *(unsigned short *)&tty->termios.c_iflag = tmp_termio.c_iflag;
    *(unsigned short *)&tty->termios.c_oflag = tmp_termio.c_oflag;
    *(unsigned short *)&tty->termios.c_cflag = tmp_termio.c_cflag;
    *(unsigned short *)&tty->termios.c_lflag = tmp_termio.c_lflag;
    tty->termios.c_line = tmp_termio.c_line;
    for(i = 0; i < NCC; i++)
        tty->termios.c_cc[i] = tmp_termio.c_cc[i];
    change_speed(tty);
    
    return 0;
}
```

#### tty 终端设备输入输出控制函数 tty_ioctl()

> 似乎与标准的 `ioctl()` 函数的参数相同
>
> * `int dev` - 设备号
> * `int cmd` - 命令号
> * `int arg` - 操作参数的指针

该函数首先根据参数中的设备号找出对应终端的 tty 结构

然后根据控制命令 cmd 分别进行处理

```c
int tty_ioctl(int dev, int cmd, int ary)
{
    struct tty_struct * tty;
    int pgrp;
    
    if (MAJOR(dev) == 5) {
        // 控制终端
        dev = current->tty;
        if (dev < 0)
            panic("tty_ioctl: dev<0");
    } else
        // 子设备号
        dev = MINOR(dev);
    
    // dev == 0，正在使用前台终端
    // 直接使用终端号 fg_console
    // dev > 0 - 虚拟终端 或 串行终端/伪终端
    tty = tty_table + (dev ? ((dev < 64) ? dev-1 : dev) : fg_console);
    
    switch (cmd) {
        case TCGETS:
            // 取终端 termios 结构体信息
            return get_termios(tty, (struct termios *) arg);
        case TCSETSF:
            // 设置 termios 结构体之前，清空读队列
            flush(tty->read_q); // 继续执行
        case TCSETWS:
            // 等待写队列中的数据处理完毕
            wait_until_sent(tty); // 继续执行
        case TCSETS:
            // 设置终端 termios 结构体
            return set_termios(tty, (struct termios *) arg, dev);
        case TCGETA:
            // 取终端的 termio 结构体
            return get_termio(tty, (struct termio *) arg);
        case TCSETAF:
            flush(tty->read_q); // 继续执行
        case TCSETAW:
            wait_until_sent(tty); // 继续执行
        case TCSETA:
            // 设置终端的 termio 结构体
            return set_termio(tty, (struct termio *) arg, dev);
        case TCSBRK:
            // 参数为 0，等待写队列处理完毕，并发送 break
            if (!arg) {
                wait_until_sent(tty);
                send_break(tty);
            }
            return 0;
        case TCXONC:
            // 开始/停止流控制
            switch (arg) {
                case TCOOFF:
                    // 挂起输出
                    tty->stopped = 1; // 停止终端输出
                    tty->write(tty); // 写缓冲队列输出
                    return 0;
                case TCOON:
                    // 恢复挂起的输出
                    tty->stopped = 0; // 恢复终端输出
                    tty->write(tty);
                    return 0;
                case TCIOFF:
                    // 终端停止输入
                    if (STOP_CHAR(tty))
                        // 放入 STOP 字符
                        PUTCH(STOP_CHAR(tty), tty->write_q);
                    return 0;
                case TCION:
                    if (START_CHAR(tty))
                        PUTCH(START_CHAR(tty), tty->write_q);
                    return 0;
            }
            return -INVAL; // 未实现
        case TCFLSH:
            // 刷新队列
            if (arg == 0)
                flush(tty->read_q);
            else if (arg == 1)
                flush(tty->write_q);
            else if (arg == 2) {
                flush(tty->read_q);
                flush(tty->write_q);
            } else
                return -EINVAL;
            return 0;
        case TIOCEXCL:
            return -EINVAL; // 未实现
        case TIOCNXCL:
            return -EINVAL; // 未实现
        case TIOCSCTTY:
            return -EINVAL; // 未实现
        case TSICGPGRP:
            // 读取前台进程组号
            verify_area((void *) arg, 4);
            put_fs_long(tty->pgrp, (unsigned long *) arg);
            return 0;
        case TIOCSPGRP:
            // 设置终端进程组号
            if ((current->tty < 0) ||
                (current->tty != dev) ||
                (tty->session != current->session))
                // 进程必须有控制终端
                return -ENOTTY;
            pgrp = get_fs_long((unsigned long *) arg);
            if (pgrp < 0)
                // 无效组号
                return -EINVAL;
            if (session_of_pgrp(pgrp) != current->session)
                // 会话与当前会话不同
                return -EPERM;
            tty->pgrp = pgrp;
            return 0;
        case TIOCOUTQ:
            // 返回写队列中还未送出的字符数
            verify_area((void *) arg, 4);
            put_fs_long(CHARS(tty->write_q), (unsigned long *) arg);
            return 0;
        case TIOCINQ:
            // 返回辅助队列中还未读取的字符数
            verify_area((void *) arg, 4);
            put_fs_long(CHARS(tty->secondary), (unsigned long *) arg);
            return 0;
        case TIOCSTI:
            // 模拟终端输入操作
            return -EINVAL; // 未实现
        case TIOCGWINSZ:
            // 读取终端设备的窗口大小信息
            return -EINVAL; // 未实现
        case TIOCSWINSZ:
            // 设置终端设备的窗口大小信息
            return -EINVAL; // 未实现
        // case ... 都未实现
        default:
            return -EINVAL;
    }
}
```

---

## Summary

这一章算是结束了

这章中的内容和文件系统结合较为紧密

所以接下来要看一看文件系统

---

