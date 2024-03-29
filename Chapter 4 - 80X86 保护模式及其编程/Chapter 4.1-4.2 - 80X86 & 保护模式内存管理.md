# Chapter 4.1-4.2 -  80X86 系统寄存器和系统指令 & 保护模式内存管理

Created by : Mr Dk.

2019 / 07 / 24 20:59

Nanjing, Jiangsu, China

---

## 4.1 80X86 系统寄存器和系统指令

协助处理器执行初始化，控制系统操作。80X86 处理器提供了：

* 标志寄存器 EFLAGS
  * 通用状态标志
  * 系统标志
    * 控制任务切换
    * 中断处理
    * 指令跟踪
    * 访问权限
* 系统寄存器
  * 内存管理 - 分段、分页机制中系统表的及地址
  * 控制处理器操作 - 比特标志位

### 4.1.1 标志寄存器

通用标志：CF、PF、AF、ZF、SF、DF、OF 等数学运算标志

系统标志 - 只允许 OS 代码修改这些标志

* TF (Trap Flag) - 启动单步执行，CPU 会在每条指令执行后产生一个调试异常
* IOPL (I/O Previlege Level) - I/O 特权级，任务的 CPL 必须低于 IOPL 才能访问 I/O 地址空间
* NT (Nested Task) - 嵌套任务
* RF (Resume Flag) - 控制 CPU 对断点指令的响应
* VM - 切换至虚拟-8086 模式

### 4.1.2 内存管理寄存器

内存管理寄存器用于指定 **内存分段管理所用系统表** 的 **基地址**，它们都是段基址寄存器：

* 全局描述符表寄存器 GDTR
  * 内容：
    * 全局描述符表 GDT 的 32-bit **线性** 基地址
    * GDT 的 16-bit 表长度
  * 指令 `LGDT` 和 `SGDT` 用于加载和保存该寄存器
  * 保护模式初始化过程中需要给 GDTR 加载新值
* 中断描述符表寄存器 IDTR
  * 内容：
    * 中断描述符表 IDT 的 32-bit **线性** 基地址
    * IDT 的 16-bit 表长度
  * 指令 `LIDT` 和 `SIDT` 用于加载和保存该寄存器
* 局部描述符表寄存器 LDTR
  * 内容：
    * LDT 的 16-bit 段选择符
    * 局部描述符表 LDT 的 32-bit **线性** 基地址
    * 16-bit 段长度
    * 描述符属性值
  * 包含 LDT 表的段必须在 GDT 表中有一个段描述符项
  * `LLDT` 和 `SLDT` 用于加载和保存基地址、段长度和描述符属性值
  * 载入段选择符时，CPU 会自动加载段描述符
  * 任务切换时，CPU 会将新任务 LDT 的段选择符和段描述符加载进 LDTR
* 任务寄存器 TR
  * 内容：
    * 当前任务 TSS 段的 16-bit 段选择符
    * 32-bit 基地址
    * 16-bit 段长度
    * 描述符属性值
  * 引用 GDT 表中的一个 TSS 类型的描述符
  * `LTR` & `STR`
  * 载入段选择符时，段描述符会被自动加载
  * 任务切换时，新任务的 TSS 段会被自动加载

### 4.1.3 控制寄存器

控制和确定 CPU 的操作模式和执行任务的特性。

#### CR0

* 协处理器控制位
* 保护控制位
  * PE - Protection Enable
    * 开启段级保护
    * 实地址模式 → 保护模式
  * PG - Paging
    * 开启分页机制
    * 禁止分页时，线性地址 = 物理地址
  * WP - Write Protect
    * 禁止超级用户程序向用户级只读页面执行写操作

改变 PE 和 PG 位时需要小心，只有执行程序有部分代码和数据在线性地址空间和物理地址空间中具有相同地址时，才能改变 PG 位的设置 - 这部分在分页和未分页的世界之间起桥梁作用。无论是否开启分页机制，这部分代码都具有相同的地址。

#### CR1

保留不用。

#### CR2

出现页异常时存放出错信息：引起异常的线性地址。

#### CR3

* 页目录表页面的物理地址
* 也称 PDBR, Page-Derectory Base address Register

由于页目录表页面是 **页对齐** 的

* 高 20-bit 有效
* 低 12-bit 保留供更高级处理器使用 (因此必须置 0)

为减少地址转换所需要的总线周期数，最近访问的页目录和页表会被存放在 CPU 的 TLB (Translation Lookaside Buffer) 中。

### 4.1.4 系统指令

用于加载系统寄存器、管理中断等。大多数系统指令只能由处于特权级 0 的 OS 软件执行。

---

## 4.2 保护模式内存管理

### 4.2.1 内存寻址

最小数据类型的寻址是对 1 字节 数据的定位。80X86 使用了一种基于段 (Segment) 的寻址技术，因此寻址需要：

* 段起始地址：由 16-bit 的段选择符指定，其中 14-bit 是地址 - 即可以选择 `2^14` 个段
* 段内偏移：由 32-bit 的偏移量决定，一个段的最大长度为 `2^32` B

因此程序中由 16-bit 的段选择符和 32-bit 的偏移构成 48-bit 的逻辑地址 (虚拟地址)。80X86 提供了 6 个专门存放 **段选择符** 的段寄存器

* CS - 寻址代码段 - `CS:[EIP]` - 当前代码段
* SS - 寻址堆栈段 - `SS:[ESP]` - 当前堆栈段
* DS - 默认数据段寄存器

指令寻址方式：偏移地址 = 基地址 + (变址 × 比例因子) + 偏移量。

### 4.2.2 地址变换

内存管理中的两个关键部分：

* 保护，防止一个任务访问另一个任务或 OS 的内存区域
* 地址变换，使 OS 在给任务分配内存时具有灵活性，使某些物理地址不被任何逻辑地址映射

变换或映射以内存块为操作单位：

* 分段
* 分页

分段和分页操作都是用驻留在内存中的表来指定变换信息。这些表只能由 OS 访问，防止应用程序修改。

80X86 的地址变换：

* 第一阶段 - 用分段机制，把逻辑地址变化为线性地址 - (即 CPU 可寻址的内存空间)
* 第二阶段 (可选) - 用分页机制，把线性地址映射到物理地址 - (即 CPU 在地址总线上能够产生的地址范围)

若不开启分页机制，则线性地址直接映射到物理地址。

#### 1. 分段机制

将 CPU 可寻址的线性地址空间划分为一些较小的段，系统中所有的段都在 CPU 的线性地址空间中。为了定位某字节，程序需要提供 48-bit 的逻辑地址：

* 段选择符
* 偏移量

段选择符提供了 **段描述符表** 中该段的 **段描述符** 的偏移量，段描述符指明：

* 段大小
* 段基地址
* 段访问权限和特权级
* 段类型

段基地址 + 逻辑地址偏移量 就可以定位段中的某个字节，即线性地址。

逻辑地址最多可包含 `2^14` (16K) 个段，每个段最长可达 4GB。虚拟地址空间容量为 `2^46` (64TB)，而线性地址空间是 CPU 可寻址空间 - 4GB。物理地址空间 - 4GB。

> 物理地址空间可以更小吗？

#### 2. 分页机制

多任务系统定义：线性地址空间比物理内存容量大很多，需要 “虚拟化” 线性地址空间 - 虚拟存储技术：产生内存空间比实际物理内存容量大很多的错觉，由分页机制支持。

将大容量线性地址空间用 **小块物理内存 + 外部存储空间** 模拟。分页时，每个段被划分为页，存储于：

* 物理内存中
* 磁盘上

OS 维护 **页目录表** 和 **页表** 来维护存放信息。当程序试图访问线性地址空间中的地址时，CPU 使用页目录表和页表将线性地址转换为物理地址。如果要访问的页不在物理内存中，则触发中断，将硬盘上的页面读入内存。对应用程序来说，OS 使页面在物理内存和硬盘之间的交换是透明的：

* 分页机制使用大小固定的内存块，适合管理物理内存，页表保存在物理地址空间
* 分段机制使用大小可变的内存块，更适合处理复杂系统的逻辑分区，段表存储在线性地址空间

### 4.2.3 保护

#### 第一类保护 - 给每个任务不同的逻辑地址空间来隔离任务

给予每个任务不同的逻辑地址到物理地址的映射。一个任务不可能生成映射到其它任务对应物理内存的逻辑地址，从而隔离。每个任务都有自己的段表和页表。切换任务的关键 - 切换新任务的地址变换表。

在所有任务中安排相同的从虚拟地址到物理地址的映射部分，将 OS 存储在公共的虚拟空间中，OS 可以被所有任务共享。

所有任务中相同的虚拟地址空间：全局地址空间 Global Address Space。每个任务对相同全局地址空间的访问将会被映射到相同的物理地址，为公共代码和数据的共享提供支持。

每个任务唯一的虚拟地址空间：局部地址空间 Local Address Space。每个任务对 **相同局部地址空间** 的访问将被映射到 **不同的物理地址**。因此 OS 可以给每个任务分配相同的虚拟地址空间，但仍然能隔绝每个任务。

#### 第二类保护 - 特权级保护

在任务中定义了 4 个执行特权级，表示任务中程序的受信程度，段中数据的敏感度。限制任务中对各个段的访问：0 - 3 表示，0 是最高特权级，3 是最低特权级。每个内存段都与一个特权级相关联

当前代码段的特权级：Current Privilege Level, CPL。确定了哪些段能够被程序访问。程序访问一个段时，比较 CPL 和被访问段的特权级，从而确定访问许可。

