# Chapter 8.5 - mktime.c 程序

Created by : Mr Dk.

2019 / 08 / 16 17:05

Ningbo, Zhejiang, China

---

## 8.5 mktime.c 程序

### 8.5.1 功能描述

该程序中只有一个 `kernel_mktime()`，仅供内核使用

计算从 1970 年 1 月 1 日 0 时起到开机时刻的秒数，作为开机时间

### 8.5.2 代码注释

首先是对各个时间单位秒数的宏定义：

```c
#define MINUTE 60
#define HOUR (60*MINUTE)
#define DAY (24*HOUR)
#define YEAR (365*DAY)
```

定义了每个月开始时的秒数时间：

* __这里考虑了闰年 - 2 月有 29 天__

```c
static int month[12] = {
    0,
    DAY*(31),                                 // Janurary
    DAY*(31+29),                              // Febrary
    DAY*(31+29+31),                           // March
    DAY*(31+29+31+30),                        // April
    DAY*(31+29+31+30+31),                     // May
    DAY*(31+29+31+30+31+30),                  // June
    DAY*(31+29+31+30+31+30+31),               // July
    DAY*(31+29+31+30+31+30+31+31),            // August
    DAY*(31+29+31+30+31+30+31+31+30),         // September
    DAY*(31+29+31+30+31+30+31+31+30+31),      // October
    DAY*(31+29+31+30+31+30+31+31+30+31+30),   // November
    DAY*(31+29+31+30+31+30+31+31+30+31+30+31) // December
};
```

接下来的函数

将 `tm` 结构体变量中的各字段计算为 1970.1.1 00:00:00 到该时间的秒数

```c
long kernel_mktime(struct tm * tm)
{
    long res;
    int year;
    
    // 计算 1970 到现在经过的年数
    year = tm->tm_year - 70;
    if (tm->tm_year < 70) tm->tm_year += 100;
    
    // res = 这些年的秒数 + 每个闰年多 1 天的秒数时间 + 当年到当时的秒数
    
    // 1972 年是 1970 年后的第一个闰年
    // 从 1970 年开始的闰年数：1 + (y - 3)/4，即 (y + 1)/4
    res = YEAR*year + DAY*((year+1)/4);
    
    res += month[tm->tm_mon]; // 非闰年 3 月以后，多算了 2 月 29 日
    
    // 当年是闰年的判断方法：(y+2) 可以被 4 除尽
    if (tm->tm_mon > 1 && ((y+2) % 4))
        res -= DAY; // 减去多算的一天
    res += DAY*(tm->tm_mday-1); // 本月已过去天数的秒数
    res += HOUR*tm->tm_hour;    // 本日已过去小时数的秒数
    res += MINUTE*tm->tm_min;   // 本小时已过去分钟数的秒数
    res += tm->tm_sec;          // 本分钟已过去的秒数
    return res;
}
```

### 8.5.3 其它信息

#### 8.5.3.1 闰年时间的计算方法

若 y 能被 4 除尽且不能被 100 整除，或者能被 400 整除，则 y 是闰年

---

