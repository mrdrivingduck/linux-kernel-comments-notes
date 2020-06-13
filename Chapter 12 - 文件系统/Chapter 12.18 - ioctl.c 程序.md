# Chapter 12.18 - ioctl.c 程序

Created by : Mr Dk.

2019 / 09 / 21 22:56

Nanjing, Jiangsu, China

---

## 12.18 ioctl.c 程序

### 12.18.1 功能描述

实现了 I/O 控制系统调用 `ioctl()`。可看作是各个具体设备驱动程序的接口函数，该函数将调用指定的文件句柄对应设备的驱动程序。

### 12.18.2 代码注释

```c
static ioctl_ptr ioctl_table[] = {
    NULL, // no dev
    NULL, // /dev/mem
    NULL, // /dev/fd
    NULL, // /dev/hd
    tty_ioctl, // /dev/ttyx
    tty_ioctl, // /dev/tty
    NULL, // dev/lp
    NULL // named pipes
};

// 设备表中的设备种数
#define NR_DEVS ((sizeof (ioctl_table)) / (sizeof (ioctl_ptr)))
```

#### sys_ioctl() - I/O 控制系统调用

```c
int sys_ioctl(unsigned int fd, unsigned int cmd, unsigned long arg)
{
    struct file * filp;
    int dev, mode;
    
    // 判断文件句柄有效性
    if (fd >= NR_OPEN || !(filp = current->filp[fd]))
        return -EBADF;
    // 文件结构对应管道 inode
    if (filp->f_inode->i_pipe)
        return (filp->f_mode & 1) ? pipe_ioctl(filp->f_inode, cmd, arg) : -EBADF;
    // 取文件属性
    mode = filp->f_inode->i_mode;
    if (!S_ISCHR(mode) && !S_ISBLK(mode))
        // 文件不是字符设备文件，也不是块设备文件
        return -EINVAL;
    dev = filp->f_inode->i_zone[0]; // 取设备号
    if (MAJOR(dev) >= NRDEVS)
        // 设备号大于系统现有设备数
        return -ENODEV;
    if (!ioctl_table[MAJOR(dev)]) // 查得对应设备的 ioctl 函数指针
        return -ENOTTY;
    return ioctl_table[MAJOR(dev)](dev, cmd, arg);
}
```

其中，最后的函数调用定义如下：

```c
typedef int (*ioctl_ptr) (int dev, int cmd, int arg);
```

即，定义了函数指针的参数列表。在 `ioctl_table` 表中找到对应设备的函数指针，传入参数并调用。

---

