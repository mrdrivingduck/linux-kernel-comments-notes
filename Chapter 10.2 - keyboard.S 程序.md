# Chapter 10.2 - keyboard.S 程序

Created by : Mr Dk.

2019 / 08 / 26 11:02

Ningbo, Zhejiang, China

---

## 10.2 keyboard.S 程序

### 10.2.1 功能描述

该汇编程序主要包含 __键盘中断处理程序__

该程序首先根据键盘上的 __特殊键__ 的状态，设置状态标志变量 mode 的值

根据引起键盘中断的 __按键扫描码__，调用已编成跳转表的扫描码处理子程序

把扫描码对应的字符放入 __读缓冲队列__ 中

接下来调用 C 函数 `do_tty_interrupt()`，在其中调用 `copy_to_block()` 行规则函数

该函数把读缓冲队列中的字符经过适当处理后，放入 __辅助队列__ 中

如果相应终端设置了回显标志，还要把字符放入 __写缓冲队列__ 中，并调用 tty 写函数

### 10.2.2 代码注释

#### 一些常量的定义

```assembly
size = 1024 # 缓冲队列字节长度

# tty_queue 结构体中各字段的偏移
head = 4
tail = 8
proc_list = 12
buf = 16
```

#### 标志字节

```assembly
mode: .byte 0 # 键盘特殊键的按下状态标志 ctrl/alt/caps/shift
leds: .byte 2 # 键盘指示灯状态标志 num-lock/caps/...
e0: .byte 0 # 扫描前导码标志，扫描码为 0xe0/0xe1
```

#### 键盘中断处理程序

键盘控制器收到用户的按键操作时

向中断控制器发出键盘中断请求信号 IRQ1

CPU 响应该请求后，进入键盘中断处理程序

该程序从键盘控制器端口 0x60 读入键盘扫描码

并调用对应的扫描码子程序进行处理

1. 若当前扫描码为 0xe0 或 0xe1，就立刻对键盘控制器作出应答，并向中断控制器发送 EOI 信号
2. 否则根据扫描码调用对应的按键处理子程序，把扫描码放入读缓冲队列中
3. 对键盘控制器作出应答，发送 EOI 信号
4. 调用 `do_tty_interrupt()`，将读缓冲队列的字符放到辅助队列中

```assembly
_keyboard_interrupt:
    pushl %eax
    pushl %ebx
    pushl %ecx
    pushl %edx
    push %ds
    push %es
    movl $0x10, %eax # 内核数据段
    mov %ax, %ds
    mov %ax, %es
    movl _blankinterval, %eax
    movl %eax, _blankcount # 预置黑屏时间计数值为 blankinterval (滴答数)
    xorl %eax, %eax # %eax is scan code
    inb $0x60, %al # 读取扫描码
    cmpb $0xe0, %al # 扫描码是 0xe0，跳转
    je set_e0
    cmpb $0xe1, %al # 扫描码是 0xe1，跳转
    je set_e1
    call key_table(, %eax, 4) # 按键处理程序 key_table + eax * 4
    movb $0, e0 # 复位 e0 标志

# 对键盘电路进行复位处理
# 首先禁止键盘，然后立刻重新允许键盘工作
e0_e1:
    inb $0x61, %al # 取 PPI 端口 B 状态，其位 7 用于允许/禁止 (0/1) 键盘
    jmp 1f # 延迟一会
1:
    jmp 1f
1:
    orb $0x80, %al # al位 7 置位 (禁止键盘工作)
    jmp 1f
1:
    jmp 1f
1:
    outb %al, $0x61 # 使 PPI PB7 位置位。
    jmp 1f
1:
    jmp 1f
1:
    andb $0x7F, %al # al 位 7 复位。
    outb %al, $0x61 # 使 PPI PB7 位复位 (允许键盘工作)
    movb $0x20, %al # 向 8259 中断芯片发送 EOI (中断结束) 信号
    outb %al, $0x20
    pushl $0 # 控制台 tty 号 =0 ，作为参数入栈
    call _do_tty_interrupt # 将收到数据转换成规范模式并存放在规范字符缓冲队列中
    addl $4, %esp # 丢弃入栈的参数，弹出保留的寄存器，并中断返回
    pop %es
    pop %ds
    popl %edx
    popl %ecx
    popl %ebx
    popl %eax
    iret
set_e0:
    movb $1, e0 # 收到扫描前导码 0xe0 时，设置 e0 标志的位 0
    jmp e0_e1
set_e1:
    movb $2, e0 # 收到扫描前导码 0xe1 时，设置 e0 标志的位 1
    jmp e0_e1
```

#### 将字符加入缓冲队列

操作范围是：`%ebx:%eax`，最多 8 个字符

顺序：

* %al
* %ah
* %eal
* %eah
* %bl
* %bh
* %ebl
* %ebh

直至 %eax 为 0

首先从 table_list 中取得控制台的读缓冲队列的地址

将 al 寄存器中的字符复制到头指针处，头指针前移

如果头指针超出缓冲区末端，就回绕到缓冲区开始处

如果缓冲区已满，就将剩余字符抛弃

如果缓冲区未满，则将 ebx 到 eax 联合右移 8B，并重复对 al 的操作过程

所有字符处理完后，保存头指针 (寄存器 → 内存)

检查是否有进程等待读队列，如有，则唤醒

```assembly
put_queue:
    pushl %ecx
    pushl %edx
    movl _table_list, %edx # 取控制台 tty 结构中读缓冲队列指针
    movl head(%edx), %ecx # 取队列头指针至 ecx
1:
    # 这部分用于将字符放入缓冲队列
    # 可以被循环使用
    movb %al, buf(%edx, %ecx) # 将 al 中的字符放入头指针位置处
    incl %ecx # 头指针前移1字节
    andl $size-1, %ecx # 头指针若超出缓冲区末端则绕回开始处。
    cmpl tail(%edx), %ecx # buffer full - discard everything
    je 3f # 队列已满，则后面未放入的字符全抛弃
    shrdl $8, %ebx, %eax # 将 ebx 中 8 个比特右移 8 位到 eax 中，ebx不变
    je 2f # 还有字符吗？若没有则跳转
    shrl $8,%ebx # 将ebx值右移8位，并跳转到标号1继续操作
    jmp 1b
2:
    # 这部分用于操作内存中的 tty_queue 结构体
    # 保存头指针，设置等待进程状态
    # 在字符全部被放入队列后被调用
    movl %ecx, head(%edx) # 若已将所有字符都放入队列，则保存头指针
    movl proc_list(%edx), %ecx # 该队列的等待进程指针
    testl %ecx, %ecx # 检测是否有等待该队列的进程。
    je 3f # 无，则跳转
    movl $0, (%ecx) # 有，则唤醒进程 (置该进程为就绪状态)
3:
    # 这部分用于结束操作
    # 或在操作过程中，队列已满后被调用
    popl %edx
    popl %ecx
    ret
```

#### 各按键或送键的处理子程序

最终将各子程序的指针放置到 `key_table` 中

供上述程序调用

---

