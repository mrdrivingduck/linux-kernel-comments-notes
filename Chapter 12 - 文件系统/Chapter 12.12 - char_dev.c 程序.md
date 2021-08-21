# Chapter 12.12 - char_dev.c 程序

Created by : Mr Dk.

2019 / 09 / 15 20:13

Nanjing, Jiangsu, China

---

## 12.12 char_dev.c 程序

### 12.12.1 功能描述

包含各个字符设备文件的访问函数，另还有一个设备读写函数指针表。

### 12.12.2 代码注释

#### rw_ttyx() - 串口终端读写操作函数

```c
static int rw_ttyx(int rw, unsigned minor, char * buf, int count, off_t * pos)
{
    return ((rw == READ) ?
            tty_read(minor, buf, count) :
            tty_write(minor, buf, count));
}
```

#### rw_tty() - 控制台终端读写操作函数

基本同上，但需要对进程是否有 **控制终端** 进行检测。

```c
static int rw_tty(int rw, unsigned minor, char * buf, int count, off_t * pos)
{
    if (current->tty < 0)
        return -EPERM;
    return rw_ttyx(rw, current->tty, buf, count, pos);
}
```

#### rw_ram() - 内存数据读写函数 - 未实现

#### rw_mem() - 物理内存数据读写函数 - 未实现

#### rw_kmem() - 内核虚拟内存数据读写函数 - 未实现

#### rw_port() - 端口读写操作函数

`pos` 为端口地址

```c
static int rw_port(int rw, char * buf, int count, off_t * pos)
{
    int i = *pos;
    
    // 端口地址小于 64k
    // 还有数据需要操作
    while (count-- > 0 && i < 65536) {
        if (rw == READ)
            put_fs_byte(inb(i), buf++);
        else
            outb(get_fs_byte(buf++), i);
        i++; // ??
    }
    
    i -= *pos; // i 为读/写的字节数
    *pos += i;
    return i;
}
```

#### rw_memory() - 内存读写操作函数

```c
static int rw_memory(int rw, unsigned minor, char * buf, int count, off_t * pos)
{
    // 内存主设备号为 1
    // 根据内存设备的子设备号
    // 调用不同的内存读写函数
    switch(minor) {
        case 0: // /dev/ram0 或 /dev/ramdisk
            return rw_ram(rw, buf, count, pos);
        case 1: // /dev/ram1 或 /dev/mem 或 ram
            return rw_mem(rw, buf, count, pos);
        case 2: // /dev/ram2 或 /dev/kmem
            return rw_kmem(rw, buf, count, pos);
        case 3: // /dev/null
            return (rw == READ) ? 0 : count;
        case 4: // /dev/port
            return rw_port(rw, buf, count, pos);
        default:
            return -EIO;
    }
}
```

#### rw_char() - 字符设备读写操作函数

根据设备的主设备号，分别调用对应的 **字符设备读写函数指针表** 中的函数。字符设备读写函数指针的定义：

```c
typedef (*crw_ptr) (int rw, unsigned minor, char * buf, int count, off_t * pos);
```

字符设备读写函数指针表：

```c
static crw_ptr crw_table[] = {
    NULL, // /dev/null
    rw_memory, // /dev/mem
    NULL, // /dev/fd
    NULL, // /dev/hd
    rw_ttyx, // /dev/ttyx
    rw_tty, // /dev/tty
    NULL, // /dev/lp
    NULL // 未命名管道
};
```

系统中的设备种数：

```c
#define NRDEVS ((sizeof(crw_table)) / (sizeof(crw_ptr)))
```

字符设备读写操作函数：

```c
int rw_char(int rw, int dev, char * buf, int count, off_t * pos)
{
    crw_ptr call_addr;
    
    if (MAJOR(dev) >= NRDEVS)
        // 设备号超出系统设备数
        return -ENODEV;
    if (!(call_addr = crw_table[MAJOR(dev)]))
        // 设备没有读写函数
        return -ENODEV;
    return call_addr(rw, MINOR(dev), buf, count, pos);
}
```

---

## Summary

从层次上来看，在对字符设备的上层调用中，需要提供设备号。接口函数根据设备的 **主设备号**，找到对应设备的读写函数指针，然后将 **子设备号** 传入各类设备的读写子函数中，在读写子函数中进入不同的分支逻辑。

