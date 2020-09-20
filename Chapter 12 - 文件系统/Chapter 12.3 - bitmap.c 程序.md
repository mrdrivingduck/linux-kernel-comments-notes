# Chapter 12.3 - bitmap.c 程序

Created by : Mr Dk.

2019 / 09 / 03 14:46

Nanjing, Jiangsu, China

---

## 12.3 bitmap.c 程序

根据文件系统中 **逻辑块** 和 **inode** 的使用情况

对逻辑块和 inode 的 bitmap 进行 bit 位的占用 / 释放操作

* `free_block()` - 释放指定设备 dev 上的逻辑块 block
  * 在高速缓冲中查找，若此块已在高速缓冲中，则释放
  * 计算逻辑块号，在逻辑块 bitmap 中将对应的 bit 位复位
  * 在包含逻辑块 bitmap 的缓冲区中，设置该缓冲块的已修改标志
* `new_block()` - 向设备 dev 申请一个逻辑块，返回逻辑块号
  * 对 dev 的逻辑块 bitmap 进行搜索，寻找首个为 0 的 bit 位
  * 置 1 占用该逻辑块，将包含该 bit 位的逻辑块 bitmap 所在缓冲块的已修改标志置位
  * 在高速缓冲区中申请相应的缓冲块，将该缓冲块清零
  * 设置缓冲块的已更新和已修改标志，释放该缓冲块
* `free_inode()` - 释放指定的 inode
  * 复位 inode bitmap 的对应 bit 位
* `new_inode()` - 为设备 dev 建立一个新的 inode
  * 从内存 inode 表中获取一个空闲表项
  * 从 inode bitmap 中找到一个空闲的 inode

### 12.3.2 代码注释

#### clear_block(addr) - 将指定地址处 1024B 的内存清零

```c
#define clear_block(addr) \
__asm__("cld\n\t" \
        "rep\n\t" \
        "stosl" \
        ::"a"(0),"c"(BLOCK_SIZE/4),"D"((long)(addr)):"cx","di")
```

#### set_bit(nr, addr) - 把指定地址开始的第 nr 个 bit 置位

```c
#define set_bit(nr,addr) ({ \
register int res __asm__("ax"); \
__asm__ __volatile__("btsl %2,%3\n\tsetb %%al": \
"=a"(res):""(0),"r"(nr),"m"(*(addr))); \
res;})
```

#### clear_bit(nr, addr) - 复位指定地址开始的第 nr 个 bit

```c
#define clear_bit(nr,addr) ({ \
register int res __asm__("ax"); \
__asm__ __volatile__("btrl %2,%3\n\tsetnb %%al": \
"=a"(res):""(0),"r"(nr),"m"(*(addr))); \
res;})
```

#### find_first_zero(addr) - 从 addr 开始寻找第 1 个 0 值 bit 位

```c
#define find_first_zero(addr) ({ \
int __res;
__asm__("cld\n" \
        "l:\tlodsl\n\t" \
        "notl %%eax\n\t" \
        "bsfl %%eax,%%edx\n\t" \
        "je 2f\n\t" \
        "addl %%edx,%%ecx\n\t" \
        "jmp 3f\n" \
        "2:\taddl $32,%%ecx\n\t" \
        "cmpl $8192,%%ecx\n\t" \
        "jl 1b\n" \
        "3:" \
        :"=c"(__res):"c"(0),"S"(addr):"ax","dx","si"); \
__res;})
```

#### free_block() - 释放逻辑块

```c
int free_block(int dev, int block)
{
    struct super_block * sb;
    struct buffer_head * bh;
    
    // 取 super block 的信息
    // 数据区开始块号、逻辑块总数信息
    if (!(sb = get_super(dev)))
        // super block 不存在
        panic("trying to free block on nonexistent device");
    if (block < sb->s_firstdatazone || b >= sb->s_nzones)
        // 逻辑块号小于盘上数据区的第 1 个逻辑块号
        // 逻辑块号大于设备上总逻辑块数
        panic("trying to free block not in datazone");
    
    bh = get_hash_table(dev, block);
    if (bh) {
        // hash 表中存在该块数据
        if (bh->b_count > 1) {
            // 该块还有人用
            brelse(bh);
            return 0;
        }
        bh->b_dirt = 0;
        bh->b_uptodate = 0;
        if (bh->b_count)
            // b_count == 1，释放
            brelse(bh);
    }
    // block 转换为从数据区开始算起的逻辑块号 (从 1 开始)
    block -= sb->s_firstdatazone - 1;
    if (clear_bit(block & 8191, sb->s_zmap[block/8192]->b_data)) {
        // block & 8191 得到 block 在 bitmap 当前块中偏移的位置
        // block / 8192 得到 block 在 bitmap 的哪个块上
        printk("block (%04x:%d) ", dev, block+sb->firstdatazone - 1);
        printk("free_block: bit already cleared\n");
    }
    // 逻辑块 bitmap 所在缓冲块的已修改标志置位
    sb->s_zmap[block/8192]->b_dirt = 1;
    return 1;
}
```

#### new_block() - 申请逻辑块

在逻辑块 bitmap 中寻找第一个 0 值 bit 位 (空闲逻辑块)，占用该逻辑块，为该逻辑块在缓冲区中取得一块缓冲块。将缓冲块清零，并设置已更新和已修改标志，返回逻辑块号。

```c
int new_block(int dev)
{
    struct buffer_head * bh;
    struct super_block * sb;
    int i, j;
    
    // 获取 super block
    if (!(sb = get_super(dev)))
        panic("trying to get new block from nonexistant device");
    // 扫描 FS 的 8 个逻辑块 bitmap
    // 寻找首个 0 值 bit - 寻找空闲逻辑块
    j = 8192;
    for (i = 0; i < 8; i++)
        if (bh = sb->s_zmap[i]) // 放置逻辑块 bitmap 的缓冲区
            if ((j = find_first_zero(bh->b_data)) < 8192)
                break;
    if (i >= 8 || !bh || j >= 8192)
        // 没有找到空闲逻辑块
        return 0;
    if (set_bit(j, bh->b_data))
        panic("new_block: bit already set");
    bh->b_dirt = 1; // 逻辑块 bitmap 所在缓冲区被修改
    
    j += i * 8192 + sb->s_firstdatazone - 1;
    if (j >= sb->s_nzones)
        // 新逻辑块号超出设备上的总逻辑块数
        return 0;
    
    // 为该逻辑块分配高速缓冲块
    if (!(bh = getblk(dev, j)))
        panic("new_block: cannot get block");
    if (bh->b_count != 1)
        // 该缓冲块块已被引用
        panic("new block: count is != 1");
    clear_block(bh->b_data); // 清空缓冲块的数据
    bh->b_uptodate = 1;
    bh->b_dirt = 1;
    brelse(bh); // 释放缓冲块
    return j;
}
```

#### free_inode() - 释放 inode

判断给定 inode 的有效性和可释放型，若 inode 正在被使用，则不能被释放。利用超级块信息对 inode bitmap 进行操作，并清空 inode 结构。

```c
void free_inode(struct m_inode * inode)
{
    struct super_block * sb;
    struct buffer_head * bh;
    
    if (!inode)
        // inode 为空
        return;
    if (!inode->i_dev) {
        // inode 未被使用
        // 清空 inode 结构
        memset(inode, 0, sizeof(*inode));
        return;
    }
    
    if (inode->i_count > 1) {
        // inode 正被其它结构使用
        // 内核有问题，停机
        printk("trying to free inode with count=%d\n", inode_i_count);
        panic("free_inode");
    }
    if (inode->i_nlinks)
        // 文件链接数不为 0
        panic("trying to free inode with links");
    
    // 取得所在设备的超级块
    if (!(sb = get_super(inode->i_dev)))
        // 超级块不存在
        panic("trying to free inode on nonexistent device");
    if (inode->i_num < 1 || inode->i_num > sb->s_ninodes)
        // inode 编号不合法
        panic("trying to free inode 0 or nonexistant inode");
    if (!(bh = sb->s_imap[inode->i_num >> 13])) // inode->i_num / 8192
        // inode 所在的 bitmap 不存在
        panic("nonexistent imap in superblock");
    // bh 指向 inode bitmap 缓冲区
    if (clear_bit(inode->i_num & 8191, bh->b_data))
        printk("free_inode: bit already cleared.\n\r");
    bh->b_dirt = 1; // inode bitmap 缓冲区已被修改
    memset(inode, 0, sizeof(*inode)); // 清空 inode 结构
}
```

#### new_inode() - 申请 inode

```c
struct m_inode * new_inode(int dev)
{
    struct m_inode * inode;
    struct super_block * sb;
    struct buffer_head * bh;
    int i, j;
    
    // 从内存 inode 表中获取一个空闲表项
    if (!(inode = get_empty_inode()))
        return NULL;
    if (!(sb = get_super(dev)))
        panic("new_inode with unknown device");
    // 扫描包含 inode bitmap 的 8 个逻辑块，寻找空闲 bit
    j = 8192;
    for (i = 0; i < 8; i++)
        if (bh = sb->s_imap[i])
            if ((j = find_first_zero(bh->b_data)) < 8192)
                break;
    if (!bh || j >= 8192 || j + i * 8192 > sb->s_ninodes) {
        // bitmap 所在的缓冲块无效，或 inode 超出范围
        iput(inode); // 放回申请的 inode
        return NULL;
    }
    
    // j 为可使用的 inode 节点号
    // bh 为对应 inode 的 bitmap 缓冲区
    // 在 bitmap 中占用该 inode
    if (set_bit(j, bh->b_data))
        panic("new_inode: bit already set");
    bh->b_dirt = 1; // inode bitmap 所在缓冲块已修改标志置位
    
    inode->i_count = 1; // 引用计数
    inode->i_nlinks = 1; // 文件链接数
    inode->i_dev = dev;
    inode->i_uid = current->euid;
    inode->i_gid = current->egid;
    inode->i_dirt = 1;
    inode->i_num = j + i * 8192; // inode 号
    inode->i_mtime = inode->i_atime = inode->i_ctime = CURRENT_TIME;
    return inode;
}
```

---

## Summary

程序是无法对磁盘上的文件系统进行 **直接操作** 的，必须要通过 **缓冲区管理程序** 进行。所以，如果要申请 inode 或逻辑块，就必须对在缓冲区中的 bitmap 映像进行操作，还要在缓冲区中分配新磁盘块的缓冲区并清零，再通过缓冲区同步机制写回磁盘。同样，在释放过程中，也要对缓冲区中的 bitmap 映像进行操作，还要清除缓冲区中已缓存的磁盘块。

---

