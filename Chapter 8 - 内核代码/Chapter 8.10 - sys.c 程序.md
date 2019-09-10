# Chapter 8.10 - sys.c 程序

Created by : Mr Dk.

2019 / 08 / 21 15:30

Ningbo, Zhejiang, China

---

## 8.10 sys.c 程序

### 8.10.1 功能描述

该程序中包含有很多系统调用功能的实现函数

其中返回值为 `-ENOSYS` 的表示本版 Linux 内核中还没有实现

程序中包含很多关于 ID 的操作

一个用户有用户 ID (uid) 和用户组 ID (gid)，是 passwd 文件中对该用户设置的 ID

在每个文件的 inode 中，都保存着宿主的 uid 和 gid

指明了文件拥有者和所属用户组，用于访问或执行文件时的权限判别

另外，在进程的任务数据结构中，保存了三种用户 ID 和组 ID

| Type    | UID                                                          | GID                                                          |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 进程 ID | uid - 拥有该进程的用户                                       | gid - 拥有该进程的用户组                                     |
| 有效 ID | euid - 指明访问文件的权限                                    | egid - 指明访问文件的权限                                    |
| 保存 ID | suid - 当执行文件的 `set-user-ID` 置位时，suid 保存着执行文件的 uid；否则 suid 等于进程的 euid | sgid - 当执行文件的 `set-group-ID` 置位时，sgid 中保存着执行文件的 gid；否则 sgid 等于进程的 egid |

suid 和 sgid 用于进程访问设置了 `set-user-ID` 或 `set-group-ID` 标志的文件

* 当执行程序时，euid 通常就是 uid，因此进程只能访问进程的有效用户规定的文件
* 如果文件的 `set-user-ID` 置位，那么进程的 euid 会被设置为该文件宿主的 uid，因此进程可以访问设置了这种标志的受限文件

比如 Linux 中的 `passwd` 命令，允许普通用户修改自己的口令 - 需要超级用户的权限

### 8.10.2 代码注释

#### 系统调用 sys_ftime()

#### 系统调用 sys_break()

#### 系统调用 sys_ptrace()

#### 系统调用 sys_gtty()

#### 系统调用 sys_rename()

#### 系统调用 sys_prof()

#### 系统调用 sys_setregid()

设置当前任务的 gid、egid

```c
int sys_setregid(int rgid, int egid)
{
    if (rgid > 0) {
        if ((current->gid == rgid) || suser())
            current->gid = rgid;
        else
            // 没有超级用户特权
            return (-EPERM);
    }
    if (egid > 0) {
        if ((current->gid == egid) || (current->egid == egid) || suser()) {
            current->egid = egid;
            current->sgid = egid;
        } else
            // 没有超级用户特权
            return (-EPERM);
    }
    return 0;
}
```

#### 系统调用 sys_setreuid()

设置任务的 uid 和 euid

```c
int sys_setreuid(int ruid, int euid)
{
    int old_ruid = current->uid;
    
    if (ruid > 0) {
        if ((current->euid == ruid) || (old_ruid == ruid) || suser())
            current->uid = ruid;
        else
            return (-EPERM);
    }
    if (euid > 0) {
        if ((old_ruid == euid) || (current->euid == euid) || suser()) {
            current->euid = euid;
            current->suid = euid;
        } else {
            current->uid = old_ruid;
            return (-EPERM);
        }
    }
    return 0;
}
```

#### 系统调用 sys_setgid()

设置进程组号

```c
int sys_setgid(int gid)
{
    if (suser())
        // 超级用户
        current->gid = current->egid = current->sgid = gid;
    else if ((gid == current->gid) || (gid == current->sgid))
        current->egid = gid;
    else
        return -EPERM;
    return 0;
}
```

#### 系统调用 sys_setuid()

设置任务的 uid

```c
int sys_setuid(int uid)
{
    if (suser())
        current->uid = current->euid = current->suid = uid;
    else if ((uid == current->uid) || (uid == current->suid))
        current->euid = uid;
    else
        return -EPERM;
    return (0);
}
```

#### 系统调用 sys_phys()

#### 系统调用 sys_lock()

#### 系统调用 sys_mpx()

#### 系统调用 sys_ulimit()

#### 系统调用 sys_time()

返回 1970.1.1 00:00:00 开始计时的秒数

```c
int sys_time(long * tloc)
{
    int i;
    
    i = CURRENT_TIME;
    if (tloc) {
        verify_area(tloc, 4);
        put_fs_long(i, (unsigned long *) tloc);
    }
    return i;
}
```

#### 系统调用 sys_stime()

设置系统开机时间

参数是从 1970.1.1 00:00:00 开始的秒数

调用进程必须具有超级用户权限

```c
int sys_stime(long * tptr)
{
    if (!suser())
        return -EPERM;
    
    // 函数提供的当前时间 - 已经运行的时间 == 新开机时间
    startup_time = get_fs_long((unsigned long *) tptr) - jiffies/HZ;
    jiffies_offset = 0;
    return 0;
}
```

#### 系统调用 sys_times()

获取当前任务运行时间统计值

```c
int sys_times(struct tms * tbuf)
{
    if (tbuf) {
        verify_area(tbuf, sizeof *tbuf);
        put_fs_long(current->utime, (unsigned long *) &tbuf->tms_utime);
        put_fs_long(current->stime, (unsigned long *) &tbuf->tms_stime);
        put_fs_long(current->cutime, (unsigned long *) &tbuf->tms_cutime);
        put_fs_long(current->cstime, (unsigned long *) &tbuf->tms_cstime);
    }
    return jiffies;
}
```

#### 系统调用 sys_brk()

设置程序数据段末尾位置

* 参数数值合理
* 系统确实有足够的内存
* 进程没有超越其最大数据段大小

数据段需要大于代码段结尾，小于堆栈结尾

这个函数不会被用户直接调用，而是由 libc 库函数进行包装

```c
int sys_brk(unsigned long end_data_seg)
{
    if (end_data_seg >= current->end_code &&
        end_data_seg < current->start_stack - 16384)
        current->brk = end_data_seg;
    return current->brk;
}
```

#### 系统调用 sys_setpgid()

设置指定进程的进程组号

```c
int sys_setpgid(int pid, int pgid)
{
    int i;
    
    if (!pid)
        pid = current->pid;
    if (!pgid)
        pgid = current->pid;
    if (pgid < 0)
        return -EINVAL;
    
    for (i = 0; i < NR_TASKS; i++)
        if (task[i] && (task[i]->pid == pid) &&
            ((task[i]->p_pptr == current) || (task[i] == current))) {
            // 任务的父进程是当前进程 或 任务就是当前进程
            
            if (task[i]->leader)
                // 会话首领
                return -EPERM;
            if ((task[i]->session != current->session) ||
                ((pgid != pid) && (session_of_pgrp(pgid) != current->session)))
                // 任务的会话与当前进程不同
                // pgid 进程组所属会话号与当前进程所属会话号不同
                return -EPERM;
            // 设置进程的组号
            task[i]->pgrp = pgid;
            return 0;
        }
    // 进程不存在
    return -ESRCH;
}
```

#### 系统调用 sys_getpgrp()

返回当前进程的进程组号

```c
int sys_getpgrp(void)
{
    return current->pgrp;
}
```

#### 系统调用 sys_setsid()

#### 系统调用 sys_getgroups()

#### 系统调用 sys_setgroups()

#### 检查当前进程是否在指定的用户组中 - in_group_p()

```c
int in_group_p(gid_t grp)
{
    int i;
    
    if (grp == current->egid)
        return 1;
    for (i = 0; i < NGROUPS; i++) {
        if (current->groups[i] == NOGROUP) // 扫描完全部有效项
            break;
        if (current->groups[i] == grp) // 扫描到用户组号
            return 1;
    }
    return 0;
}
```

#### utsname 结构

```c
static struct utisname thisname = {
    UTS_SYSNAME,    // 当前操作系统名称 "Linux"
    UTS_NODENAME,   // 网络结点名称 (主机名) "(none)"
    UTS_RELEASE,    // 当前操作系统 release "0"
    UTS_VERSION,    // 操作系统版本 "0.12"
    UTS_MACHINE     // 系统运行的硬件类型名称 "i386"
};
```

#### 系统调用 sys_uname()

获取系统名称等信息

```c
int sys_uname(struct utsname * name)
{
    int i;
    
    if (!name)
        return -ERROR;
    verify_area(name, sizeof *name); // 检测用户空间内存
    for (i = 0; i < sizeof *name; i++)
        put_fs_byte(((char *) &thisname)[i], i + (char *) name);
    return 0;
}
```

#### 系统调用 sys_sethostname()

#### 系统调用 sys_getrlimit()

#### 系统调用 sys_setrlimit()

#### 系统调用 sys_getrusage()

当前进程或其子进程的资源使用情况

#### 系统调用 sys_gettimeofday()

取得系统当前时间

#### 系统调用 sys_settimeofday()

设置系统当前时间

#### 将从 CMOS 中读取的时间调整为 GMT 时间保存 adjust_clock()

```c
void adjust_clock()
{
    startup_time += sys_tz.tz_minuteswest * 60;
}
```

#### 系统调用 sys_umask()

设置当前进程创建文件属性屏蔽码，并返回原屏蔽码

```c
int sys_umask(int mask)
{
    int old = current->umask;
    
    current->umask = mask & 0777;
    return (old);
}
```

---

## Summary

这一些系统调用

大部分是对 PCB 以及一些全局数据结构进行 set 或者 get

向用户空间提供了接口，操作进程中的各种信息

---

