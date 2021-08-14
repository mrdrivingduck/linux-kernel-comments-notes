# Chapter 8.3 - traps.c 程序

Created by : Mr Dk.

2019 / 08 / 15 10:57

Ningbo, Zhejiang, China

---

## 8.3 traps.c 程序

### 8.3.1 功能描述

实现了 asm.s 汇编程序中调用的 C 函数，用于显示出错位置和出错号等调试信息。其中的 `die()` 函数用于在中断处理中显示详细的出错信息，而 `trap_init()` 函数则在之前的 `init/main.c` 中被调用，初始化硬件异常处理中断向量。

### 8.3.2 代码注释

首先，定义了三个嵌入汇编的宏函数，作用分别为：

* 取段 seg 中地址 addr 处的一个字节
* 取段 seg 中地址 addr 处的一个长字
* 取 fs 段寄存器的值

其中，定义了一个寄存器变量 `__res`，将会被保存在一个寄存器中，以便于快速访问和操作。

```c
// 取段 seg 中地址 addr 处的一个字节
#define get_seg_byte(seg, addr) ({ \
    register char __res; \
    __asm__( \
        "push %%fs; mov %%ax, %%fs; movb %%fs:%2,%%al; pop %%fs" \
        : "=a" (__res) : "0" (seg), "m" (*(addr)) \
    ); \
    __res; \
})
```

```c
// 取段 seg 中地址 addr 处的一个长字
#define get_seg_long(seg, addr) ({ \
    register long __res; \
    __asm__( \
        "push %%fs; mov %%ax, %%fs; movl %%fs:%2,%%eax; pop %%fs" \
        : "=a" (__res) : "0" (seg), "m" (*(addr)) \
    ); \
    __res; \
})
```

```c
// 取 fs 段寄存器的值
#define _fs() ({ \
    register unsigned short __res; \
    __asm__( \
        "mov %%fs, %%ax" : "=a" (__res): \
    ); \
    __res; \
})
```

接下来是一个通用的子函数 `die()`

* 用于打印错误名称、出错码、调用程序的 EIP、EFLAGS、ESP、fs 段寄存器
* 段基址、段长度、pid、任务号、10B 指令码
* 若堆栈在用户数据段，则打印 16B 堆栈内容

```c
// str - 错误名字符串
// esp_ptr - 被中断的出错程序信息在栈中信息的指针 esp0
// nr - 出错码
static void die(char *str, long esp_ptr, long nr)
{
    long *esp = (long *) esp_ptr;
    int i;
    
    printk("%s: %04x\n\r", str, nr & 0xffff);
    printk("EIP:\t%04x:%p\nEFLAGS:\t%p\nESP:\t%04x:%p\n",
            esp[1], esp[0], esp[2], esp[4], esp[3]);
    printk("fs: %04x\n", _fs());
    printk("base: %p, limit: %p\n", get_base(current->ldt[1]), get_limit(0x17));
    if (esp[4] == 0x17) {
        printk("Stack: ");
        for (i = 0; i < 10; i++)
            printk("%02x ", 0xff & get_seg_byte(esp[1], (i+(char *)esp[0])));
        printk("\n");
    }
    
    str(i); // 取当前运行任务的任务号
    printk("Pid: %d, process nr: %d\n\r", current->pid, 0xffff & i);
    
    for (i = 0; i < 10; i++)
        printk("%02x ", 0xff & get_seg_byte(esp[1], (i+(char *)esp[0])));
    printk("\n\r");
    do_exit(11);
}
```

接下来是 asm.s 中调用的中断处理 C 函数，以 `do_` 前缀开头：

```c
void do_double_fault(long esp, long error_code)
{
    die("double fault", esp, error_code);
}
```

```c
void do_general_protection(long esp, long error_code)
{
    die("general protection", esp, error_code);
}
```

```c
void do_alignment_check(long esp, long error_code)
{
    die("alignment check", esp, error_code);
}
```

```c
void do_divide_error(long esp, long error_code)
{
    die("divide error", esp, error_code);
}
```

```c
void do_int3(long *esp, long error_code,
             long fs, long es, long ds,
             long ebp, long esi, long edi,
             long edx, long ecx, long ebx, long eax)
{
    int tr;
    __asm__("str %%ax" : "=a" (tr) : "" (0)); // 取任务寄存器 TR
    printk("eax\t\tebx\t\tecx\t\tedx\n\r%8x\t%8x\t%8x\t%8x\n\r",
            eax, ebx, ecx, edx);
    printk("esi\t\tedi\t\tebp\t\tesp\n\r%8x\t%8x\t%8x\t%8x\n\r",
            esi, edi, ebp, (long) esp);
    printk("\n\rds\tes\tfs\ttr\n\r%4x\t%4x\t%4x\t%4x\n\r",
            ds, es, fs, tr);
    printk("EIP: %8x  CS: %4x  EFLAGS: %8x\n\r", esp[0], esp[1], esp[2]);
}
```

```c
void do_nmi(long esp, long error_code)
{
    die("nmi", esp, error_code);
}
```

```c
void do_debug(long esp, long error_code)
{
    die("debug", esp, error_code);
}
```

```c
void do_overflow(long esp, long error_code)
{
    die("overflow", esp, error_code);
}
```

```c
void do_bounds(long esp, long error_code)
{
    die("bounds", esp, error_code);
}
```

```c
void do_invalid_op(long esp, long error_code)
{
    die("invalid operand", esp, error_code);
}
```

```c
void do_device_not_available(long esp, long error_code)
{
    die("device not available", esp, error_code);
}
```

```c
void do_coprocessor_segment_overrun(long esp, long error_code)
{
    die("coprocessor segment overrun", esp, error_code);
}
```

```c
void do_invalid_TSS(long esp, long error_code)
{
    die("invalid TSS", esp, error_code);
}
```

```c
void do_segment_not_present(long esp, long error_code)
{
    die("segment not present", esp, error_code);
}
```

```c
void do_stack_segment(long esp, long error_code)
{
    die("stack segment", esp, error_code);
}
```

```c
void do_coprocessor_error(long esp, long error_code)
{
    if (last_task_used_math != current)
        return;
    die("coprocessor error", esp, error_code);
}
```

```c
void do_reserved(long esp, long error_code)
{
    die("reserved (15, 17-47) error", esp, error_code);
}
```

下面定义了一些函数原型，用于中断初始化：

> 没弄懂的一点是，只声明了原型，没有函数实现啊

```c
void page_exception(void);              // 页异常 (page_fault)
void divide_error(void);                // INT 0
void debug(void);                       // INT 1
void nmi(void);                         // INT 2
void int3(void);                        // INT 3
void overflow(void);                    // INT 4
void bounds(void);                      // INT 5
void invalid_op(void);                  // INT 6
void device_not_available(void);        // INT 7
void double_fault(void);                // INT 8
void coprocessor_segment_overrun(void); // INT 9
void invalid_TSS(void);                 // INT 10
void segment_not_present(void);         // INT 11
void stack_segment(void);               // INT 12
void general_protection(void);          // INT 13
void page_fault(void);                  // INT 14
void coprocessor_error(void);           // INT 16
void reserved(void);                    // INT 15
void parallel_interrupt(void);          // INT 39
void irq13(void);                       // INT 45
void alignment_check(void);             // INT 46
```

下面是内核初始化程序中调用的 `trap_init()` 函数，其中用到了 `set_trap_gate()` 和 `set_system_gate()`，都用到了 IDT 中的陷阱门：前者设置的特权级为 0，后者为 3。这两个函数是嵌入汇编程序 (`include/asm/system.h`)。

```c
void trap_init(void)
{
    int i;

    // 设置中断向量值
    set_trap_gate(0, &divide_error); 
    set_trap_gate(1, &debug);
    set_trap_gate(2, &nmi);
    set_system_gate(3, &int3); /* int3-5 can be called from all */
    set_system_gate(4, &overflow);
    set_system_gate(5, &bounds);
    set_trap_gate(6, &invalid_op);
    set_trap_gate(7, &device_not_available);
    set_trap_gate(8, &double_fault);
    set_trap_gate(9, &coprocessor_segment_overrun);
    set_trap_gate(10, &invalid_TSS);
    set_trap_gate(11, &segment_not_present);
    set_trap_gate(12, &stack_segment);
    set_trap_gate(13, &general_protection);
    set_trap_gate(14, &page_fault);
    set_trap_gate(15, &reserved);
    set_trap_gate(16, &coprocessor_error);
    set_trap_gate(17, &alignment_check);
    // 下面把 INT 17-47 的陷阱门均先设置为 reserved
    // 以后各硬件初始化时会重新设置自己的陷阱门
    for (i = 18; i < 48; i++)
        set_trap_gate(i, &reserved);
    
    // 协处理器 INT 45 的陷阱门描述符
    set_trap_gate(45, &irq13);
    // 允许 8259A 主芯片的 IRQ2 中断请求
    outb_p(inb_p(0x21) & 0xfb, 0x21);
    // 允许 8259A 从芯片的 IRQ13 中断请求
    outb(inb_p(0xA1) & 0xdf, 0xA1);
    // 设置并行口 1 的中断 0x27 陷阱门描述符
    set_trap_gate(39, &parallel_interrupt);
}
```

