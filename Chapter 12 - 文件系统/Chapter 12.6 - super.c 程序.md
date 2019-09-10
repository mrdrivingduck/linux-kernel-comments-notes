# Chapter 12.6 - super.c 程序

Created by : Mr Dk.

2019 / 09 / 06 14:03

Nanjing, Jiangsu, China

---

## 12.6 super.c 程序

### 12.6.1 功能描述

对文件系统中的超级块进行操作的函数

此外还有文件系统加载 / 卸载的系统调用

### 12.6.2 代码注释

#### Super block 的数据结构定义

磁盘上的 super block

```c
struct d_super_block {
    unsigned short s_ninodes; // inode 数量
    unsigned short s_nzones; // 逻辑块数
    unsigned short s_imap_blocks; // inode bitmap 所占的数据块数
    unsigned short s_zmap_blocks; // 逻辑块 bitmap 所占的数据块数
    unsigned short s_firstdatazone; // 第一个数据逻辑块
    unsigned short s_log_zone_size; // 每个逻辑块包含的数据块数
    unsigned long s_max_size; // 文件最大长度
    unsigned short s_magic; // 文件系统 magic
};
```

内存中的 super block

* 前面的部分和磁盘中的 super block 完全相同

```c
struct super_block {
    unsigned short s_ninodes; // inode 数量
    unsigned short s_nzones; // 逻辑块数
    unsigned short s_imap_blocks; // inode bitmap 所占的数据块数
    unsigned short s_zmap_blocks; // 逻辑块 bitmap 所占的数据块数
    unsigned short s_firstdatazone; // 第一个数据逻辑块
    unsigned short s_log_zone_size; // 每个逻辑块包含的数据块数
    unsigned long s_max_size; // 文件最大长度
    unsigned short s_magic; // 文件系统 magic
    
    struct buffer_head * s_imap[8]; // inode bitmap 所在的缓冲块指针，8 块
    struct buffer_head * s_zmap[8]; // 逻辑块 bitmap 所在的缓冲块指针，8 块
    unsigned short s_dev; // 超级块所在的设备号
    struct m_inode * s_isup; // 被安装到的文件系统根目录的 inode
    struct m_inode * s_imount; // 被安装到的 inode
    unsigned long s_time; // 修改时间
    struct task_struct * s_wait; // 等待该 super block 的进程
    unsigned char s_lock; // 被锁定标志
    unsigned char s_rd_only; // 只读标志
    unsigned char s_dirt; // 已修改标志
};
```

```c
struct super_block super_block[NR_SUPER]; // 8 项
```

#### 测试指定位偏移处 bit 位的值

* 仅测试并返回 bit 的值
* 不对 bit 进行任何改动

```c
#define set_bit(bitnr, addr) ({ \
register int __res __asm__("ax"); \
__asm__("bt %2,%3;setb %%al":"=a"(__res):"a"(0),"r"(bitnr),"m"(*addr)); \
__res; })
```

#### 锁定超级块 - lock_super()

若超级块被锁定，则将当前任务置为不可中断的等待状态

直到超级块解锁，任务被明确唤醒

```c
static void lock_super(struct super_block * sb)
{
    cli();
    while (sb->s_lock)
        sleep_on(&(sb->s_wait));
    sb->s_lock = 1;
    sti();
}
```

#### 对指定超级块解锁 - free_super()

```c
static void free_super(struct super_block * sb)
{
    cli();
    sb->s_lock = 0;
    wake_up(&(sb->s_wait));
    sti();
}
```

#### 睡眠等待超级块解锁 - wait_on_super()

```c
static void wait_on_super(struct super_block * sb)
{
    cli();
    while (sb->s_lock)
        sleep_on(&(sb->s_wait));
    sti();
}
```

#### 取指定设备的超级块 - get_super()

在内存的超级块表中搜索指定设备 dev 的超级块结构体

```c
struct super_block * get_super(int dev)
{
    struct super_block * s;
    
    if (!dev)
        return NULL;
    
    s = 0 + super_block;
    while (s < super_block + NR_SUPER)
        if (s->s_dev == dev) {
            wait_on_super(s); // 等待超级块解锁
            // 睡眠后需要重新判断
            if (s->s_dev == dev)
                return s;
            s = 0 + super_block; // 从头扫描
        } else
            s++;
    return NULL;
}
```

#### 放回指定设备的超级块 - put_super()

释放设备使用的超级块数组项 - `s_dev` 置为 0

释放设备的 inode bitmap 和逻辑块 bitmap 占用的高速缓冲

如果超级块对应的是根文件系统，或其某个 inode 上已经安装了其它文件系统，则不能释放

```c
void put_super(int dev)
{
    struct super_block * sb;
    int i;
    
    if (dev == ROOT_DEV) {
        // 根文件系统设备
        printk("root diskette changed: prepare for armageddon\n\r");
        return;
    }
    if (!(sb = get_super(dev)))
        return;
    if (sb->s_imount) {
        printk("Mounted disk changed - tssk, tssk\n\r");
        return;
    }
    
    // 找到指定设备的超级块了
    lock_super(sb); // 锁定该超级块
    sb->s_dev = 0; // 释放超级块表项
    // 释放 bitmap 缓冲区
    for (i = 0; i < I_MAP_SLOTS; i++)
        brelse(sb->s_imap[i]);
    for (i = 0; i < Z_MAP_SLOTS; i++)
        brelse(sb->s_zmap[i]);
    free_super(sb); // 超级块解锁
    return;
}
```

#### 读取指定设备的超级块 - read_super()

如果设备 dev 上的文件系统超级块已经在超级块表中，则直接返回超级块项的指针

否则从设备 dev 上读取超级块到缓冲区，再从缓冲区复制到超级块数组中

```c
static struct super_block * read_super(int dev)
{
    struct super_block * s;
    struct buffer_head * bh;
    int i, block;
    
    if (!dev)
        return NULL;
    check_disk_change(dev);
    
    if (s = get_super(dev))
        // 超级块已在超级块表中
        return s;
    // 否则在超级块表中寻找一个空闲项
    for (s = 0 + super_block; ; s++) {
        if (s >= NR_SUPER + super_block)
            return NULL; // 没找到空闲项
        if (!s->s_dev)
            break; // s_dev == 0
    }
    
    // 该超级块被用于设备 dev 上的文件系统
    s->s_dev = dev;
    s->s_isup = NULL;
    s->s_imount = NULL;
    s->s_time = 0;
    s->s_rd_only = 0;
    s->s_dirt = 0;
    
    lock_super(s); // 锁定超级块
    if (!(bh = bread(dev, 1))) {
        // 读取超级块到缓冲区中
        // 失败，则释放
        s->s_dev = 0;
        free_super(s);
        return NULL;
    }
    // 从缓冲区复制到内存中的超级块表中
    *((struct d_super_block *) s) = *((struct d_super_block *) bh->b_data);
    brelse(bh);
    
    // 检查超级块的有效性
    if (s->s_magic != SUPER_MAGIC) {
        // 不是正确的文件系统
        s->s_dev = 0;
        free_super(s);
        return NULL;
    }
    
    // 初始化内存超级块结构的 bitmap 空间
    for (i = 0; i < I_MAP_SLOTS; i++)
        s->s_imap[i] = NULL;
    for (i = 0; i < Z_MAP_SLOTS; i++)
        s->s_zmap[i] = NULL;
    // inode bitmap 保存在 2 号块开始的逻辑块中
    block = 2;
    // 读取 inode bitmap 到缓冲区中
    for (i = 0; i < s->s_imap_blocks; i++)
        if (s->s_imap[i] = bread(dev, block))
            block++;
        else
            break;
    // 读取逻辑块 bitmap 到缓冲区中
    for (i = 0; i < s->s_zmap_blocks; i++)
        if (s->s_zmap[i] = bread(dev, block))
            block++;
        else
            break;
    
    if (block != 2 + s->s_imap_blocks + s->s_zmap_blocks) {
        // 读出的 bitmap 块数不等于位图该有的逻辑块数
        // 释放所有缓冲块
        for (i = 0; i < I_MAP_SLOTS; i++)
            brelse(s->s_imap[i]);
        for (i = 0; i < Z_MAP_SLOTS; i++)
            brelse(s->s_zmap[i]);
        s->s_dev = 0; // 释放数组项
        free_super(s); // 解锁
        return NULL;
    }
    
    // 将 bitmap 的第 0 位置 1
    s->s_imap[0]->b_data[0] |= 1;
    s->s_zmap[0]->b_data[0] |= 1;
    free_super(s);
    return s;
}
```

#### 系统调用 - 卸载文件系统 - sys_umount()

根据块设备文件名，找到对应的 inode，从而获得其设备号

复位文件系统超级块的相应字段，释放 bitmap 占用的缓冲块

最后执行高速缓冲与设备上的同步操作

```c
int sys_umount(char * dev_name)
{
    struct m_inode * inode;
    struct super_block * sb;
    int dev;
    
    // 找设备对应的 inode
    if (!(inode = namei(dev_name)))
        return -ENOENT;
    dev = inode->i_zone[0]; // 设备号
    if (!(S_ISBLK(inode->i_mode))) {
        // 不是块设备
        iput(inode); // 放回 inode
        return -ENOTBLK;
    }
    
    // 已经取得设备号了，所以可以将 inode 放回
    iput(inode);
    // 检查卸载条件
    if (dev == ROOT_DEV)
        // 根文件系统不能被卸载
        return -EBUSY;
    if (!(sb = get_super(dev)) || !(sb->s_imount))
        // 超级块表中没有该超级块
        // 文件系统没有被安装
        return -ENOENT;
    if (!sb->s_imount->i_mount)
        // 安装标志未置位
        printk("Mounted inode has i_mount=0\n");
    // 扫描内存 inode 表
    for (inode = inode_table + 0; inode < inode_table + NR_INODE; inode++)
        if (inode->i_dev == dev && inode->i_count)
            // 有进程正在使用该设备上的文件
            return -EBUSY;
    
    // 卸载条件已满足
    sb->s_imount->i_mount = 0; // 被安装的 inode 复位安装标志
    iput(sb->s_imount); // 释放被安装的 inode
    sb->i_imount = NULL;
    iput(sb->s_isup); // 释放 root inode
    sb->s_isup = NULL;
    
    put_super(dev); // 释放设备上的超级块，以及 bitmap 占用的高速缓冲块
    sync_dev(dev); // 高速缓冲同步到设备
    return 0;
}
```

#### 系统调用 - 挂载文件系统 - sys_mount()

```c
int sys_mount(char * dev_name, char * dir_name, int rw_flag)
{
    struct m_inode * dev_i, * dir_i;
    struct super_block * sb;
    int dev;
    
    // 根据设备名，找到设备 inode，取得设备号
    if (!(dev_i = namei(dev_name)))
        return -ENOENT;
    dev = dev_i->i_zone[0];
    if (!S_ISBLK(dev_i->i_mode)) {
        iput(dev_i);
        return -EPERM;
    }
    
    // 已经取得设备号了，设备 inode 可以被放回了
    iput(dev_i);
    
    // 找到挂载点目录的 inode
    if (!(dir_i = namei(dir_name)))
        return -ENOENT;
    if (dir_i->i_count != 1 || dir_i->i_num == ROOT_INO) {
        // 引用计数不为 1 (应当仅在此处被引用)
        // 根文件系统也安装在这个 inode 上
        iput(dir_i);
        return -EBUSY;
    }
    if (!S_ISDIR(dir_i->i_mode)) {
        // 挂载点不是一个目录 (应当是一个目录！)
        iput(dir_i);
        return -EPERM;
    }
    
    // 设备 inode、挂载点 inode 检查完毕
    // 读取文件系统的超级块信息
    if (!(sb = read_super(dev))) {
        // 读取超级块失败
        iput(dir_i);
        return -EBUSY;
    }
    if (sb->s_imount) {
        // 文件系统已挂载到了其它地方
        iput(dir_i);
        return -EBUSY;
    }
    if (dir_i->i_mount) {
        // 将要安装到的 inode 已挂载了文件系统
        iput(dir_i);
        return -EPERM;
    }
    
    sb->s_imount = dir_i; // 文件系统挂载点指向指定目录的 inode
    dir_i->i_mount = 1; // 该目录已挂载文件系统
    dir_i->i_dirt = 1; // 挂载点目录 inode 已被修改
    return 0;
}
```

#### 挂载根文件系统 - mount_root()

在系统初始化时被调用

初始化文件表 `file_table[]` 和超级块表

读取根文件系统的超级块，并取得 root inode

统计并显示根文件系统上的可用资源

```c
void mount_root(void)
{
    int i, free;
    struct super_block * p;
    struct m_inode * mi;
    
    // 检查磁盘 inode 结构体大小
    if (32 != sizeof(struct d_inode))
        panic("bad i-node size");
    // 初始化文件表
    for (i = 0; i < NR_FILE; i++)
        file_table[i].f_count = 0;
    // 软盘......
    if (MAJOR(ROOT_DEV) == 2) {
        printk("Insert root floppy and press ENTER");
        wait_for_keypress();
    }
    // 初始化超级块表
    for (p = &super_block[0]; p < &super_block[NR_SUPER]; p++) {
        p->s_dev = 0;
        p->s_lock = 0;
        p->s_wait = NULL;
    }
    
    // 挂载根文件系统
    if (!(p = read_super(ROOT_DEV)))
        // 无法读取超级块
        panic("Unable to mount root");
    if (!(mi = iget(ROOT_DEV, ROOT_INO)))
        // 无法获取 root inode
        panic("Unable to read root i-node");
    
    mi->i_count += 3; // 被引用 4 次
    p->s_isup = p->s_imount = mi; // 超级块指向 root inode
    current->pwd = mi;
    current->root = mi; // 当前进程为 init 进程
    
    // 根文件系统的资源统计
    free = 0; // 空闲逻辑块
    i = p->s_nzones;
    while (--i >= 0)
        if (!set_bit(i & 8191, p->s_zmap[i >> 13]->b_data))
            // i 在 bitmap 中的位偏移
            // i>>13 表示 i/8192，即位于哪一个 bitmap 块中
            free++;
    printk("%d/%d free blocks\n\r", free, p->s_nzones);
    
    free = 0; // 空闲 inode
    i = p->s_ninodes + 1;
    while (--i >= 0)
        if (!set_bit(i & 8191, p->s_imap[i >> 13]->b_data))
            free++;
    printk("%d/%d free inodes\n\r", free, p->s_ninodes);
}
```

---

## Summary

关于文件系统的挂载

首先需要根据挂载的设备名，找到设备 inode，从而取得设备号 → 找到文件系统的超级块

然后根据挂载点的目录名，找到挂载点目录的 inode

挂载后，文件系统超级块将指向挂载点的 inode

而挂载点的 inode 中则被设定为已经挂载了文件系统

挂载根文件系统时

根据根设备号得到根设备的超级块

并且在初始化过程中确定了 root inode

和 root inode

将超级块设定为指向 root inode，也就是根目录，从而完成挂载

---

