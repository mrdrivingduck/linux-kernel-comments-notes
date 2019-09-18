# Chapter 12.14 - open.c 程序

Created by : Mr Dk.

2019 / 09 / 18 22:13

Nanjing, Jiangsu, China

---

## 12.14 open.c 程序

接下来的程序属于文件系统中的高级操作和管理部分

> 上一个程序中的 read、write 之类的函数
>
> 都是直接引用 inode 对文件进行操作
>
> 因为这些文件已经被打开了
>
> 而在这个程序中，需要根据设备名或文件名，
>
> 调 `namei()` 类似的函数取到对应的 inode
>
> 从而打开文件或设备，为接下来的读写操作作铺垫

### 12.14.1 功能描述

文件的创建、打开、关闭

文件宿主和属性的修改

文件访问权限的修改

文件操作时间的修改等

### 12.14.2 代码注释

#### sys_ustat() - 文件系统状态信息

```c
int sys_ustat(int dev, struct ustat * ubuf)
{
    return -ENOSYS;
}
```

#### sys_utime() - 设置文件访问和修改时间

```c
int sys_utime(char * filename, struct utimbuf * times)
{
    struct m_inode * inode;
    long actime, modtime;
    
    // 根据文件名取得对应的 inode
    if (!(inode = namei(filename)))
        // 找不到 inode
        return -ENOENT;
    // 如果 times 结构不为空，则根据 times 中的时间设置
    // 否则默认使用系统当前时间
    if (times) {
        actime = get_fs_long((unsigned long *) &times->actime);
        modtime = get_fs_long((unsigned long *) &times->modtime);
    } else {
        actime = modtime = CURRENT_TIME;
    }
    
    // 设置 inode 时间
    inode->i_atime = actime;
    inode->i_mtime = modtime;
    inode->i_dirt = 1; // inode 已被修改
    iput(inode); // 放回 inode
    return 0;
}
```

#### sys_access() - 检查文件的访问权限

POSIX 标准建议使用 ruid (真实用户 id)

如果允许访问，则返回 0

```c
int sys_access(const char * filename, int mode)
{
    struct m_inode * inode;
    int res, i_mode;
    
    // 文件的访问权限信息保存在文件的 inode 中
    // 所以要先取 inode
    mode &= 0007; // mode 的有效位为低三位
    if (!(inode = namei(filename)))
        return -EACCES; // 找不到 inode，无访问权限
    i_mode = res = inode->i_mode & 0777;
    
    if (current->uid == inode->i_uid)
        res >>= 6;
    else if (current->gid == inode->i_gid)
        res >>= 3;
    if ((res & 0007 & mode) == mode)
        return 0;
    
    iput(inode);
    
    if ((!current->uid) && (!(mode & 1) || (i_mode & 0111)))
        // 用户 ID 为 0 (super uesr)
        // 且，执行位为 0 或文件可以被任何人执行
        return 0;
    
    return -EACCES;
}
```

#### sys_chdir() - 改变工作目录

```c
int sys_chdir(const char * filename)
{
    struct m_inode * inode;
    
    if (!(inode = namei(filename)))
        // 找不到对应的目录 inode
        return -ENOENT;
    if (!S_ISDIR(inode->i_mode)) {
        // 不是目录
        iput(inode);
        return -ENOTDIR;
    }
    
    iput(current->pwd); // 放回当前进程原来的 pwd 的 inode
    current->pwd = inode;
    return 0;
}
```

#### sys_chroot() - 改变根目录

```c
int sys_chroot(const char * filename)
{
    struct m_inode * inode;
    
    if (!(inode = namei(filename)))
        // 找不到 inode
        return -ENOENT;
    if (!S_ISDIR(inode->i_mode)) {
        // 不是目录
        iput(inode);
        return -ENOTDIR;
    }
    
    iput(current->root); // 放回当前进程原来的 root 的 inode
    current->root = inode;
    return 0;
}
```

#### sys_chmod() - 修改文件属性

```c
int sys_chmod(const char * filename, int mode)
{
    struct m_inode * inode;
    
    if (!(inode = namei(filename)))
        return -ENOENT;
    if ((current->euid != inode->i_uid) && !suser()) {
        // 没有修改文件的权限，也不是超级用户
        iput(inode);
        return -EACCES;
    }
    
    // 设置文件属性
    imode->i_mode = (mode & 0777) | (inode->i_mode & ~07777);
    inode->i_dirt = 1; // inode 已被修改
    iput(inode); // 放回 inode
    return 0;
}
```

#### sys_chown() - 修改文件宿主

```c
int sys_chown(const char * filename, int uid, int gid)
{
    struct m_inode * inode;
    
    if (!(inode = namei(filename)))
        return -ENOENT;
    if (!suser()) {
        // 不是超级用户
        iput(inode);
        return -EACCES;
    }
    
    inode->i_uid = uid;
    inode->i_gid = gid;
    inode->i_dirt = 1; // inode 已修改
    iput(inode);
    return 0;
}
```

#### check_char_dev() - 检查和设置字符设备属性

如果打开的文件是 tty 终端字符设备时

对当前进程属性和系统 tty 表进行修改和设置

因此只处理主设备号为 4 (`/dev/ttyxx`) 或 5 (`/dev/tty`) 的情况

```c
static int check_char_dev(struct m_inode * inode, int dev, int flag)
{
    struct tty_struct * tty;
    int min;
    
    if (MAJOR(dev) == 4 || MAJOR(dev) == 5) {
        if (MAJOR(dev) == 5)
            min = current->tty;
        else
            min = MINOR(dev);
        if (min < 0)
            return -1;
        
        if ((IS_A_PTY_MASTER(min)) && (inode->i_count > 1))
            // 设备已被其它进程使用
            return -1;
        
        tty = TTY_TABLE(min); // 指向设备
        if (!(flag & O_NOCTTY) &&
            current->leader &&
            current->tty < 0 &&
            tty->session == 0) {
            // 允许为进程设置控制终端
            current->tty = min;
            tty->session = current->session;
            tty->pgrp = current->pgrp;
        }
        if (flag & O_NONBLOCK) {
            // 设置了非阻塞标志
            TTY_TABLE(min)->termios.c_cc[VMIN] = 0;
            TTY_TABLE(min)->termios.c_cc[VTIME] = 0;
            TTY_TALBE(min)->termios.c_lflag &= ~ICANON;
        }
    }
    
    return 0;
}
```

#### sys_open() - 打开 (创建) 文件

```c
int sys_open(const char * filename, int flag, int mode)
{
    struct m_inode * inode;
    struct file * f;
    int i, fd;
    
    mode &= 0777 & ~current->umask;
    // 在进程 PCB 的文件结构中找一个空闲项
    for (fd = 0; fd < NR_OPEN; fd++)
        if (!current->filp[fd])
            break;
    if (fd >= NR_OPEN)
        // 没找到空闲项
        return -EINVAL;
    
    // 设置当前进程的 close_on_exec bitmap
    // 每个 bit 确定了调用 execve() 时需要关闭的文件句柄
    // 当打开一个文件时，默认情况下，文件句柄在子进程中也处于打开状态
    // 因此，这里需要 reset 该 bit
    current->close_on_exec &= ~(1 << fd);
    // 在内核文件表中搜索一个空闲项
    f = file_table + 0;
    for (i = 0; i < NR_FILE; i++, f++)
        if (!f->f_count)
            break;
    if (i >= NR_FILE)
        return -EINVAL;
    
    (current->filp[fd] = f)->f_count++; // 占用文件表中的该项
    if ((i = open_namei(filename, flag, mode, &inode)) < 0) {
        // 打开失败，释放文件结构项
        current->filp[fd] = NULL;
        f->f_count = 0;
        return i;
    }
    
    if (S_ISCHR(inode->i_mode))
        // 检测字符设备文件
        if (check_char_dev(inode, inode->i_zone[0], flag)) {
            // 无法打开字符设备文件
            // 放回 inode，并置空文件结构
            iput(inode);
            current->filp[fd] = NULL;
            f->f_count = 0;
            return -EAGAIN;
        }
    if (S_ISBLK(inode->i_mode))
        // 检查盘片是否被更换
        check_disk_change(inode->i_zone[0]);
    
    // 初始化文件结构
    f->f_mode = inode->i_mode;
    f->f_flags = flag;
    f->f_count = 1;
    f->f_inode = inode;
    f->f_pos = 0;
    return fd;
}
```

#### sys_creat() - 创建文件

```c
int sys_creat(const char * pathname, int mode)
{
    return sys_open(pathname, O_CREAT | O_TRUNC, mode);
}
```

#### sys_close() - 关闭文件

```c
int sys_close(unsigned int fd)
{
    struct file * filp;
    
    if (fd >= NR_OPEN)
        return -EINVAL;
    current->close_on_exec &= ~(1 << fd); // 复位关闭文件句柄的 bitmap
    if (!(filp = current->filp[fd]))
        return -EINVAL;
    
    current->filp[fd] = NULL; // 文件结构项置空
    if (filp->f_count == 0)
        panic("Close: file count is 0");
    if (--filp->f_count) // 引用次数递减，如果还不为 0，则还有其它进程使用
        return 0;
    iput(filp->f_inode); // 文件已没有进程引用，空闲，释放 inode
    return 0;
}
```

---

## Summary

读完这个程序好像有点明白了

内核维护了一个文件结构的数组

进程 PCB 中打开的文件

实际上是指向这个数组中的数组项

进程的 PCB 中指向该项的数组下标

就是该进程对应该文件的文件描述符

---

