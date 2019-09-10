# Chapter 12.10 - file_dev.c 程序

Created by : Mr Dk.

2019 / 09 / 10 13:03

Nanjing, Jiangsu, China

---

接上节

用于操作常规文件的低级支持函数

供 `read()` 和 `write()` 系统调用使用

即系统调用 `sys_write()` 和 `sys_read()`

以及不同设备的低层支持函数：

* 访问正规文件 - `file_write()` / `file_read()`
* 访问管道文件 - `pipe_write()` / `pipe_read()`
* 访问块设备文件 - `block_write()` / `block_read()`
* 访问字符设备文件 - `rw_char()`

在系统调用中，根据参数提供的文件描述符的属性

判断出文件属于哪种类型

分别调用相应的处理函数，并进入对应的驱动程序中

## 12.10 file_dev.c 程序

### 12.10.1 功能描述

同样也是访问文件数据

但是是通过指定 __文件路径名__ 的方式进行操作

函数参数给出 __inode__ 和 __文件结构__ 的信息

* inode 用于获取相应设备号
* 文件结构用于获得文件当前的读写指针信息

而上一节 `block_dev.c` 中的函数则是直接指定了设备号和文件读写位置

### 12.10.2 代码注释

#### file_read() - 读文件函数

```c
int file_read(struct m_inode * inode, struct file * filp, char * buf, int count)
{
    int left, chars, nr;
    struct buffer_head * bh;
    
    if ((left = count) <= 0)
        // 要读取的字节小于等于 0
        return 0;
    
    while (left) {
        // 找到对应设备上文件数据块号对应的逻辑块号
        if (nr = bmap(inode, (filp->f_pos) / BLOCK_SIZE)) {
            // 若逻辑块存在，则将该逻辑块读入缓冲区
            if (!(bh = bread(inode->i_dev, nr)))
                break;
        } else
            bh = NULL;
        
        nr = filp->f_pos % BLOCK_SIZE; // 块内偏移，块内读取的首地址
        chars = MIN(BLOCK_SIZE - nr, left); // 在本块内，还需要读取的字节数
        // 从块内某个位置开始，到块结束处
        // 一整块
        // 剩余字节从块头部开始，但不满一块
        
        filp->f_pos += chars;
        left -= chars;
        
        // 复制 chars 字节到用户缓冲区中
        if (bh) {
            char * p = bh->b_data + nr; // 指向读取首地址
            while (chars-- > 0)
                // 复制数据至用户空间
                put_fs_byte(*(p++), buf++);
            brelse(bh);
        } else {
            while (chars-- > 0)
                // 如果对应块不存在，则填充 0 值
                put_fs_byte(0, buf++);
        }
        
        inode->i_atime = CURRENT_TIME; // inode 访问时间
        return (count - left) ? (count - left) : -ERROR; // 返回读取的字节数
    }
}
```

#### file_write() - 写文件函数

```c
int file_write(sturct m_inode * inode, struct file * filp, char * buf, int count)
{
    off_t pos;
    int block, c;
    struct buffer_head * bh;
    char * p;
    int i = 0;
    
    if (filp->f_flags & O_APPEND)
        // 追加模式
        // 指针指向文件尾部
        pos = inode->i_size;
    else
        // 否则指针指向文件当前的读写位置
        pos = filp->f_pos;
    
    // 已写入字节数 i
    // 待写入字节数 count
    while (i < count) {
        // 寻找 pos 所在文件数据块对应的逻辑块号
        // 如果不存在，则创建一块
        if (!(block = create_block(inode, pos / BLOCK_SIZE)))
            break;
        // 将对应逻辑块读入缓冲区
        if (!(bh = bread(inode->i_dev, block)))
            break;
        
        c = pos % BLOCK_SIZE; // 块内偏移
        p = bh->b_data + c; // 指向写入操作开始位置
        bh->b_dirt = 1;
        c = BLOCK_SIZE - c; // 本缓冲块的剩余可写字节
        if (c > count - i)
            c = count - i; // 待写数据已经写不满本块
        
        pos += c; // 调整文件指针
        if (pos > inode->i_size) {
            // 文件指针超出文件当前长度
            // 增加文件长度
            inode->i_size = pos;
            inode->i_dirt = 1;
        }
        i += c; // 累加已写入字节数
        while (c-- > 0)
            // 从用户空间拷贝数据
            *(p++) = get_fs_byte(buf++);
        brelse(bh); // 释放当前缓冲区
    }
    
    // 数据已全部写入文件
    inode->i_mtime = CURRENT_TIME; // 文件修改时间
    if (!(filp->f_flags & O_APPEND)) {
        // 调整文件读写指针至写入数据位置结束的地方
        filp->f_pos = pos;
        inode->i_ctime = CURRENT_TIME; // inode 修改时间
    }
    return (i ? i : -1); // 返回已写字节数
}
```

---

