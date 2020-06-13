# Chapter 12.16 - stat.c 程序

Created by : Mr Dk.

2019 / 09 / 21 21:26

Nanjing, Jiangsu, China

---

## 12.16 stat.c 程序

### 12.16.1 功能描述

实现获取文件状态信息的系统调用 `stat()` 和 `fstat()`，并将文件信息以 `stat` 结构体的形式返回到用户缓冲区中。`stat` 结构体中的所有字段信息都能在文件的 inode 中获得。

* `stat()` 利用文件名获取信息
* `fstat()` 使用文件句柄 (描述符) 来取得信息

```c
struct stat {
    dev_t st_dev; // 含有文件的设备号
    ino_t st_ino; // 文件的 inode 号
    umode_t st_mode; // 文件类型和属性
    nlink_t st_nlink; // 文件的链接数
    uid_t st_uid; // 文件的用户号
    gid_t st_gid; // 文件的组号
    dev_t st_rdev; // 设备号 (如果文件是设备文件)
    off_t st_size; // 文件字节大小 (文件是常规文件)
    time_t st_atime; // 最后访问时间
    time_t st_mtime; // 最后修改时间
    time_t st_ctime; // 最后结点修改时间
};
```

### 12.16.2 代码注释

#### cp_stat() - 根据 inode 获取文件状态信息

```c
static void cp_stat(struct m_inode * inode, struct stat * statbuf)
{
    struct stat tmp;
    int i;
    
    // 验证用户缓冲区大小可以放下 stat 结构体
    verify_area(statbuf, sizeof(struct stat));
    
    // 将 inode 中的信息复制到 stat 结构体
    tmp.st_dev = inode->i_dev;
    tmp.st_ino = inode->i_num;
    tmp.st_mode = inode->i_mode;
    tmp.st_nlink = inode->i_nlinks;
    tmp.st_uid = inode->i_uid;
    tmp.st_gid = inode->i_gid;
    tmp.st_rdev = inode->i_zone[0]; // 设备文件设备号
    tmp.st_size = inode->i_size;
    tmp.st_atime = inode->i_atime;
    tmp.st_mtime = inode->i_mtime;
    tmp.st_ctime = inode->i_ctime;
    
    // 将结构体拷贝到用户空间缓冲区中
    for (i = 0; i < sizeof(tmp); i++)
        put_fs_byte(((char *) &tmp)[i], (char *) statbuf + i);
}
```

#### sys_stat() - 根据文件名，获取相关文件状态信息的系统调用

```c
int sys_stat(char * filename, struct stat * statbuf)
{
    struct m_inode * inode;
    
    if (!(inode = namei(filename))) // 根据文件名找到 inode
        return -ENOENT;
    cp_stat(inode, statbuf);
    iput(inode); // 放回 inode
    return 0;
}
```

#### sys_fstat() - 根据文件句柄获取文件状态系统调用

```c
int sys_fstat(unsigned int fd, struct stat * statbuf)
{
    struct file * f;
    struct m_inode * inode;
    
    // 利用句柄取得文件 inode
    if (fd >= NR_OPEN ||
        !(f = current->filp[fd]) ||
        !(inode = f->f_inode))
        return -EBADF;
    cp_stat(inode, statbuf);
    return 0;
}
```

#### sys_lstat() - 符号链接文件状态系统调用

只取符号链接文件本身的状态，不跟随链接文件的状态。

```c
int sys_lstat(char * filename, struct stat * statbuf)
{
    struct m_inode * inode;
    
    if (!(inode = lnamei(filename))) // 取符号链接 inode，不跟随符号链接
        return -ENOENT;
    cp_stat(inode, statbuf);
    iput(inode);
    return 0;
}
```

#### sys_readlink() - 读取符号链接系统调用

读取符号链接文件中的内容，放到指定长度的用户缓冲区中 (若缓冲区太小，就会截断符号链接的内容)。

```c
int sys_readlink(const char * path, char * buf, int bufsiz)
{
    struct m_inode * inode;
    struct buffer_head * bh;
    int i;
    char c;
    
    // 用户缓冲区长度必须在 1-1023 之间
    if (bufsiz <= 0)
        return -EBADF;
    if (bufsiz > 1023)
        bufsiz = 1023;
    
    // 验证用户缓冲区大小是否足够
    verify_area(buf, bufsiz);
    
    if (!(inode = lnamei(path))) // 取符号链接的 inode
        return -ENOENT;
    // 读取符号链接文件到高速缓冲
    if (inode->i_zone[0])
        bh = bread(inode->i_dev, inode->i_zone[0]);
    else
        bh = NULL;
    iput(inode); // 放回 inode
    
    if (!bh)
        return 0;
    
    // 向用户缓冲区复制
    i = 0;
    while (i < bufsiz && (c = bh->b_data[i])) {
        i++;
        put_fs_byte(c, buf++);
    }
    brelse(bh); // 释放高速缓冲
    return i;
}
```

---

