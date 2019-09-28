# Chapter 10.4 - serial.c 程序

Created by : Mr Dk.

2019 / 08 / 26 17:18

Ningbo, Zhejiang, China

---

## 10.4 serial.c 程序

### 10.4.1 功能描述

本程序实现系统串行端口的初始化

为使用串行终端设备做好准备工作

`rs_init()` 函数

* 设置了默认的串行通信参数
* 设置串行端口的中断陷阱门

`rs_write()` 函数

* 把用于串行终端设备的写缓冲队列中的字符通过串行线路发给远端的终端设备
* 会在文件系统中，操作字符设备文件时被调用
* 实际上只是开启 __串行发送保持器已空中断标志__
  * 在 UART 将数据发送出去以后，允许发送中断信号
  * 具体的发送操作在 `rs_io.s` 程序中完成

### 10.4.2 代码注释

#### init() - 串行端口初始化

设置指定串行端口的波特率

允许除 __写保持寄存器空__ 以外的所有中断源

参数 `port` 为串行端口基地址

```c
static void init(int port)
{
    outb_p(0x80, port + 3);
    outb_p(0x30, port);
    outb_p(0x00, port + 1);
    outb_p(0x03, port + 3);
    outb_p(0x0b, port + 4);
    outb_p(0x0d, port + 1);
    (void) inb (port);
}
```

#### rs_init() - 初始化串行中断程序和串行接口

```c
void rs_init(void)
{
    set_intr_gate(0x24, rs1_interrupt); // 串口 1 的中断向量 - IRQ4
    set_intr_gate(0x23, rs2_interrupt); // 串口 2 的中断向量 - IRQ3
    init(tty_table[64].read_q->data); // 初始化串口 1
    init(tty_table[65].read_q->data); // 初始化串口 2
    outb(inb_p(0x21) & 0xE7, 0x21); // 允许 8259A 响应 IRQ3、IRQ4
}
```

#### rs_write() - 串行接口写函数

该函数在 `tty_write()` 已将数据放入写缓冲队列后被调用

在该程序中，首先检查写队列是否为空，然后设置相应的中断寄存器

该函数 __仅开启发送保持寄存器已空中断的标志__

以下过程由串口中断处理函数完成：

发送保持寄存器为空时，UART 就会产生中断

在中断处理程序中，取出写缓冲队列尾部的字符，输出到发送保持寄存器中

发送出去后，发送保持寄存器变空，从而再次引发中断请求

直到所有字符被发送

此时，程序重新禁止发送保持寄存器已空时发出的中断

```c
void rs_write(struct tty_struct * tty)
{
    cli(); // 关中断
    if (!EMPTY(tty->write_q))
        // 写队列不为空
        // 读取中断允许寄存器，置位允许中断标志后，写回该寄存器
        outb(inb_p(tty->write_q->data+1) | 0x02, tty->write_q->data+1);
    sti(); // 开中断
}
```

---

