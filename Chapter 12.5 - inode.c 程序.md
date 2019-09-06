# Chapter 12.5 - inode.c 程序

Created by : Mr Dk.

2019 / 09 / 05 16:10

Nanjing, Jiangsu, China

---

## 12.5 inode.c 程序

### 12.5.1 功能描述

`iget()` 函数

* 从设备 dev 上读取指定结点号 nr 的 inode
* 并把 inode 的引用计数字段值 `i_count` 增 1

`iput()` 函数

* 把 inode 引用计数值减 1
* 若 inode 的链接计数为 0，则释放 inode 占用的所有磁盘逻辑块

在某时刻，进程不需要持续使用一个 inode 时就应当调用 `iput()` 函数

内核代码通常调用 `iput()` 的时机：

* inode 的引用计数字段增 1
* 调用了 `namei()`、`dir_namei()` 或 `open_namei()` 函数
* 调用了 `iget()`、`new_inode()` 或 `get_empty_inode()` 函数
* 关闭文件，且没有其它进程使用该文件
* 卸载文件系统

`bmap()` 函数

用于把文件数据块映射到对应的盘块上

参数为 inode 指针和文件中的数据块号

在对应文件数据块不存在的情况下

根据 create 标志判断是否需要在盘上建立盘块

并返回在对应设备上的盘块号

函数主要对 inode 中的 `i_zone[]` 进行处理

根据 `i_zone[]` 中设置的盘块号，设置逻辑块 bitmap 的占用情况

### 12.5.2 代码注释

#### inode 的数据结构

磁盘上的 inode (32B)

```c
struct d_inode {
    unsigned short i_mode; // 文件类型和属性
    unsigned short i_uid; // 用户 id
    unsigned long i_size; // 文件大小
    unsigned long i_time; // 修改时间
    unsigned char i_gid; // 组 id
    unsigned char i_nlinks; // 链接计数 (多少个目录指向该 inode)
    unsigned short i_zone[9]; // 逻辑块号 (直接、一次间接、二次间接)
};
```

内存中的 inode - 其中前 7 项与 d_node 完全一样

```c
struct m_inode {
    unsigned short i_mode; // 文件类型和属性
    unsigned short i_uid; // 用户 id
    unsigned long i_size; // 文件大小
    unsigned long i_time; // 修改时间
    unsigned char i_gid; // 组 id
    unsigned char i_nlinks; // 链接计数 (多少个目录指向该 inode)
    unsigned short i_zone[9]; // 逻辑块号 (直接、一次间接、二次间接)
    
    struct task_struct * i_wait; // 等待 inode 的进程
    struct task_struct * i_wait2; // for pipes
    unsigned long i_atime; // 最后访问时间
    unsigned long i_ctime; // 修改时间
    unsigned short i_dev; // inode 所在设备号
    unsigned short i_num; // inode 号
    unsigned short i_count; // inode 被使用次数 (0 为空闲)
    unsigned char i_lock; // 锁定标志
    unsigned char i_dirt; // 修改标志
    unsigned char i_pipe; // 管道标志
    unsigned char i_mount; // 安装标志
    unsigned char i_seek; // 搜寻标志
    unsigned char i_update; // 更新标志
};
```

内存中的 inode 表：

```c
struct m_inode inode_table[NR_INODE] = { { 0, }, }; // 32 项
```

#### 等待 inode 可用 - wait_on_inode()

```c
static inline void wait_on_buffer(struct m_inode * inode)
{
    cli();
    while (inode->i_lock)
        sleep_on(&inode->i_wait);
    sti();
}
```

#### 对 inode 上锁 - lock_inode()

```c
static inline void lock_inode(struct m_inode * inode)
{
    cli();
    while (inode->i_lock)
        sleep_on(&inode->i_wait);
    inode->i_lock = 1;
    sti();
}
```

#### 对 inode 解锁 - unlock_inode()

```c
static inline void unlock_inode(struct m_inode * inode)
{
    inode->i_lock = 0;
    wake_up(&inode->i_wait);
}
```

#### 释放设备 dev 在内存 inode 表中的所有 inode

```c
void invalidate_inodes(int dev)
{
    int i;
    struct m_inode * inode;
    
    // 扫描内存 inode 表中每一项
    inode = 0 + inode_table;
    for (i = 0; i < NR_INODE; i++, inode++) {
        wait_on_inode(inode); // 等待 inode 解锁
        if (inode->i_dev == dev) {
            if (inode->i_count)
                printk("inode in use on removed disk\n\r");
            inode->i_dev = inode->i_dirt = 0; // 释放 inode
        }
    }
}
```

#### 同步 inode - sync_inodes()

将内存 inode 表中所有的 inode 与设备上的 inode 作同步操作

```c
void sync_inodes(void)
{
    int i;
    struct m_inode * inode;
    
    // 扫描内存 inode 表
    inode = 0 + inode_table;
    for (i = 0; i < NR_INODE; i++, inode++) {
        wait_on_inode(inode);
        if (inode->i_dirt && !inode->i_pipe)
            // inode 已修改，且不是管道结点
            write_inode(inode); // 写入缓冲区
    }
}
```

#### 文件数据块与盘块的映射 - _bmap()

将指定的文件数据块对应到设备的逻辑块上，并返回逻辑块号

* 如果创建标志置位，那么对应逻辑块不存在时就申请新逻辑块
* 返回文件数据块对应在设备上的逻辑块号

```c
static int _bmap(struct m_inode * inode, int block, int create)
{
    struct buffer_head * bh;
    int i;
    
    // 判断参数有效性
    if (block < 0)
        // 文件数据块号小于 0
        panic("_bmap: block<0");
    if (block >= 7 + 512 + 512 * 512)
        // 文件数据块号大于文件系统表示范围
        panic("_bmap: block>big");
    
    // 直接块
    if (block < 7) {
        if (create && !inode->i_zone[block])
            // 创建标志置位
            // 数据块对应的逻辑块为空
            // 向设备申请一块磁盘块
            if (inode->i_zone[block] = new_block(inode->i_dev)) {
                inode->i_ctime = CURRENT_TIME;
                inode->i_dirt = 1; // inode 已被修改
            }
        return inode->i_zone[block];
    }
    
    // 一次间接块
    block -= 7;
    if (block < 512) {
        if (create && !inode->i_zone[7])
            // 创建标志置位
            // 一次间接块对应逻辑块为空
            if (inode->i_zone[7] = new_block(inode->i_dev)) {
                inode->i_dirt = 1;
                inode->i_ctime = CURRENT_TIME;
            }
        if (!inode->i_zone[7])
            // 申请磁盘块失败
            // 或 inode 没有一次间接块，映射失败
            return 0;
        // 将一次间接块读入缓冲区
        if (!(bh = bread(inode->i_dev, inode->i_zone[7])))
            return 0;
        // 一次间接块中，文件数据块对应的逻辑块
        i = ((unsigned short *) (bh->b_data)) [block];
        if (create && !i)
            // 创建标志置位 && 文件数据块没有对应逻辑块
            if (i = new_block(inode->i_dev)) {
                // 申请逻辑块，将逻辑块号写入缓冲区中的一次间接块中
                // 缓冲区修改标志置位
                ((unsigned short *) (bh->b_data)) [block] = i;
                bh->b_dirt = 1;
            }
        brelse(bh); // 释放缓冲区
        return i;
    }
    
    // 二次间接块
    block -= 512;
    if (create && !inode->i_zone[8])
        // 二次间接块对应逻辑块为空
        if (inode->i_zone[8] = new_block(inode->i_dev)) {
            // 申请二次间接块
            inode->i_dirt = 1;
            inode->i_ctime = CURRENT_TIME;
        }
    if (!inode->i_zone[8])
        // 申请失败 || 二次间接块本来就为空
        return 0;
    if (!(bh = bread(inode->i_dev, inode->i_zone[8])))
        // 为二次间接块申请缓冲区
        return 0;
    i = ((unsigned short *) bh->b_data) [block >> 9]; // 第 block / 512 项 (第 block/512 个一次间接块)
    if (create && !i)
        if (i = new_block(inode->i_dev)) {
            // 申请一次间接块
            ((unsigned short *) (bh->b_data)) [block >> 9] = i;
            bh->b_dirt = 1; // 二次间接块的缓冲区被修改
        }
    brelse(bh); // 释放二次间接块的缓冲区
    if (!i)
        // 一次间接块不存在
        return 0;
    if (!(bh = bread(inode->i_dev, i)))
        // 为一次间接块申请缓冲区
        return 0;
    i = ((unsigned short *) bh->b_data) [block & 511]; // 逻辑块在一次间接块中的位置
    if (create && !i)
        // 逻辑块不存在，申请逻辑块
        if (i = new_block(inode->i_dev)) {
            // 一次间接块对应的缓冲区被修改
            ((unsigned short *) (bh->b_data)) [block & 511] = i;
            bh->b_dirt = 1;
        }
    brelse(bh); // 释放一次间接块对应的缓冲区
    return i;
}
```

#### 取文件数据块在设备上的对应逻辑块号 - bmap()

```c
int bmap(struct m_inode * inode, int block)
{
    return _bmap(inode, block, 0); // 逻辑块不存在时，不申请
}
```

#### 取文件数据块在设备上对应的逻辑块号，若不存在就创建一块 - create_block()

```c
int create_block(struct m_inode * inode, int block)
{
    return _bmap(inode, block, 1);
}
```

#### 放回一个 inode - iput()

将 inode 的引用计数递减

* 管道 inode - 唤醒等待的进程
* 块设备 inode - 刷新设备

```c
void iput(struct m_inode * inode)
{
    if (!inode)
        return;
    wait_on_inode(inode);
    if (!inode->i_count)
        panic("iput: trying to free free inode");
    if (inode->i_pipe) {
        wake_up(&inode->i_wait);
        wake_up(&inode->i_wait2);
        if (--inode->i_count)
            // inode 还有引用，则直接返回
            return;
        // inode 已无更多引用
        // 释放管道占用的内存页面
        free_page(inode->i_size);
        // inode 变为空闲
        inode->i_count = 0;
        inode->i_dirt = 0;
        inode->i_pipe = 0;
        return;
    }
    if (!inode->i_dev) {
        // 设备号为 0
        inode->i_count--;
        return;
    }
    if (S_ISBLK(inode->i_mode)) {
        // 块设备
        sync_dev(inode->i_zone[0]); // 设备号
        wait_on_inode(inode);
    }
    
repeat:
    if (inode->i_count > 1) {
        // inode 还有人再用，不能释放
        inode->i_count--;
        return;
    }
    // i_count == 1
    if (!inode->i_nlinks) {
        // inode 链接数为 0，对应的文件被删除
        // 释放该 inode 的所有的逻辑块
        truncate(inode);
        free_inode(inode);
        return;
    }
    if (inode->i_dirt) {
        write_inode(inode);
        wait_on_inode(inode);
        goto repeat; // 因为睡眠了，所以要重复判断
    }
    
    inode->i_count--; // == 0，已释放
    return;
}
```

#### 从 inode 表中获取一个空闲 inode 项 - get_empty_inode()

寻找引用计数 count 为 0 的 inode，将其写盘后清零

引用计数被置 1

```c
struct m_inode * get_empty_inode(void)
{
    struct m_inode * inode;
    static struct m_inode * last_inode = inode_table;
    int i;
    
    do {
        inode = NULL;
        for (i = NR_INODE; i; i--) {
            if (++last_inode >= inode_table + NR_INODE)
                last_inode = inode_table; // 返回 inode 表头
            if (!last_inode->i_count) {
                inode = last_inode;
                if (!inode->i_dirt && !inode->i_lock)
                    // 可以使用该 inode
                    break;
            }
        }
        if (!inode) {
            // 没有找到空闲 inode 结点
            // 打印 inode 表供调试使用，停机
            for (i = 0; i < NR_INODE; i++)
                printk("%04x: %6d\t", inode_table[i].i_dev, inode_table[i].i_num);
            panic("No free inodes in mem");
        }
        wait_on_inode(inode); // 等待该 inode 解锁
        while (inode->i_dirt) {
            write_inode(inode); // 同步 inode
            wait_on_inode(inode); // 等待解锁
        }
    } while (inode->i_count); // 如果 inode 又被占用，则又要重新寻找
    
    memset(inode, 0, sizeof(*inode)); // inode 清零
    inode->i_count = 1;
    return inode;
}
```

#### 获取管道 inode - get_pipe_inode()

扫描 inode 表，取得一个空闲 inode

取得一页空闲内存供管道使用

将得到 inode 的引用计数置位 2 (Reader + Writer)

```c
struct m_inode * get_pipe_inode(void)
{
    struct m_inode * inode;
    
    if (!(inode = get_empty_inode()))
        return NULL;
    if (!(inode->i_size = get_free_page())) {
        // i_size 字段指向缓冲区
        inode->i_count = 0;
        return NULL;
    }
    inode->i_count = 2;
    PIPE_HEAD(*inode) = PIPE_TAIL(*inode) = 0; // 管道头尾指针
    inode->i_pipe = 1; // 管道标志
    return inode;
}
```

#### 取得一个 inode - iget()

从设备读取指定编号的 inode 到内存 inode 表中

返回该 inode 的指针

```c
struct m_inode * iget(int dev, int nr)
{
    struct m_inode * inode, * empty;
    
    if (!dev)
        panic("iget with dev==0");
    empty = get_empty_inode();
    
    // inode 可能已在 inode 表中
    inode = inode_table;
    while (inode < NR_INODE + inode_table) {
        if (inode->i_dev != dev || inode->i_num != nr) {
            inode++;
            continue;
        }
        wait_on_inode(inode); // 等待 inode 解锁
        if (inode->i_dev != dev || inode->i_num != nr) {
            inode = inode_table; // 重头扫描
            continue;
        }
        inode->i_count++;
        if (inode->i_mount) {
            int i;
            
            for (i = 0; i < NR_SUPER; i++)
                if (super_block[i].s_imount == inode)
                    break;
            if (i >= NR_SUPER) {
                printk("Mounted inode hasn't got sb\n");
                if (empty)
                    iput(empty);
                return inode;
            }
            iput(inode);
            dev = super_block[i].s_dev;
            nr = ROOT_INO;
            inode = inode_table;
            continue;
        }
        
        if (empty)
            iput(empty);
        return inode;
    }
    
    // 在 inode 表中没有找到指定的 inode
    // 利用之前申请的空闲 inode 在 inode 表中建立该 inode
    if (!empty)
        return NULL;
    
    inode = empty;
    inode->i_dev = dev;
    inode->i_num = nr;
    read_inode(inode);
    return inode;
}
```

#### 读取指定的 inode 信息 - read_inode()

从设备上读取含有指定 inode 信息的盘块，读入缓冲区

从缓冲区复制到指定的 inode 结构中

```c
static void read_inode(struct m_inode * inode)
{
    struct super_block * sb;
    struct buffer_head * bh;
    int block;
    
    lock_inode(inode);
    // 取得设备 super block
    if (!(sb = get_super(inode->i_dev)))
        panic("trying to read inode without dev");
    // 取得 inode 编号对应的逻辑块号
    // 启动块 + 超级块 + inode bitmap 占用块 + 逻辑块 bitmap 占用块
    // + (inode_num - 1) / 每块含有的 inode 号
    block = 2 + sb->s_imap_blocks + sb->s_zmap_blocks + 
        (inode->i_num-1)/INODE_PER_BLOCK;
    // 取得对应缓冲区
    if (!(bh = bread(inode->i_dev, block)))
        panic("unable to read i-node block");
    // 从缓冲区指定项中复制数据
    *(struct d_inode *) inode = ((struct d_inode *) bh->b_data) [(inode->i_num-1) % INODES_PER_BLOCK];
    brelse(bh); // 释放缓冲区
    if (S_ISBLK(inode->i_mode)) {
        // 块设备文件
        int i = inode->i_zone[0]; // 设备号
        if (blk_size[MAJOR(i)])
            inode->i_size = 1024 * blk_size[MAJOR(i)][MINOR(i)];
        else
            inode->i_size = 0x7fffffff;
    }
    
    unlock_inode(inode);
}
```

#### 将 inode 写入缓冲区中 - write_inode()

```c
static void write_inode(struct m_inode * inode)
{
    struct super_block * sb;
    struct buffer_head * bh;
    int block;
    
    lock_inode(inode); // 锁定 inode
    if (!inode->i_dirt || !inode->i_dev) {
        // inode 未修改 || inode 设备号为 0
        // 不需修改，直接退出
        unlock_inode(inode);
        return;
    }
    if (!(sb = get_super(inode->i_dev)))
        panic("trying to write inode without device");
    // 计算 inode 对应的逻辑块号
    block = 2 + sb->s_imap_blocks + sb->s_zmap_blocks + (inode->i_num-1)/INODES_PER_BLOCK;
    // 将 inode 对应的逻辑块读入缓冲区
    if (!(bh = bread(inode->i_dev, block)))
        panic("unable to read i-node block");
    // 复制内存 inode 中的信息到缓冲区中
    ((struct d_inode *) bh->b_data)[(inode->i_num-1) % INODES_PER_BLOCK] = *(struct d_inode *) inode;
    bh->b_dirt = 1; // 缓冲区已修改
    inode->i_dirt = 0; // inode 已同步到缓冲区中
    brelse(bh); // 释放缓冲区
    unlock_inode(inode);
}
```

---

## Summary

之前一直没弄清楚的是磁盘、缓冲区、内存三个位置的数据结构之间的关系

这三个地方的数据是独立的

inode 表位于内存中

这个源代码文件中，基本上是对内存中的 inode 表进行操作

而 inode 实际上位于磁盘

在读取 inode 时，需要先将 inode 所在的磁盘逻辑块读进缓冲区

然后再从缓冲区中将数据拷贝到内存的 inode 表中

而写入 inode 时，需要将内存 inode 表中的数据拷贝到对应的缓冲区中

再从缓冲区同步到磁盘上

对于内存中的 inode 来说，写入 inode 时

只需要同步到缓冲区中，即可认为同步完成

从缓冲区到磁盘的同步由 __缓冲区管理机制__ 负责完成

---

