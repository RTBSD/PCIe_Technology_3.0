# 第4章：地址空间与事务路由

> 来源：《PCI Express Technology 3.0》
> 原文章节：Chapter 4: Address Space & Transaction Routing

---

## 4.1 地址空间概述

PCIe继承了PCI的三种地址空间模型，并进行了扩展以支持更高的性能和更大的地址范围。

### 4.1.1 PCIe地址空间类型

PCIe定义了三种独立的地址空间：

1. **内存空间（Memory Space）**
   - 用于内存映射I/O（MMIO）
   - 支持32位和64位地址
   - 主要的数据传输机制

2. **I/O空间（I/O Space）**
   - 用于遗留兼容性
   - 仅支持32位地址
   - 在原生PCIe设备中已弃用

3. **配置空间（Configuration Space）**
   - 用于设备配置和枚举
   - 每个功能4KB（256字节PCI兼容 + 3840字节扩展）
   - 只能通过配置事务访问

**图4-1：PCIe地址空间映射**

![地址空间](../images/图4-1.地址空间.png)

### 4.1.2 地址空间特性对比

| 特性 | 内存空间 | I/O空间 | 配置空间 |
|-----|---------|---------|---------|
| 地址宽度 | 32/64位 | 32位 | 通过ID路由 |
| 主要用途 | 数据传输 | 遗留兼容 | 设备配置 |
| 缓存 | 可缓存 | 不可缓存 | 不可缓存 |
| 预取 | 支持 | 不支持 | 不支持 |
| 原生PCIe | 主要机制 | 已弃用 | 必需 |

---

## 4.2 基址寄存器（BAR）

### 4.2.1 BAR概述

**基址寄存器（Base Address Register, BAR）**是PCIe设备用来向系统软件报告其内存和I/O空间需求的机制。每个PCIe功能可以实现多达6个BAR（Type 0头部）。

**图4-2：BAR寄存器结构**

![内存地址路由](../images/图4-2.内存地址路由.jpg)

```
Bit 31/63                              Bit 3   Bit 2-1   Bit 0
+--------------------------------------+--------+---------+------+
|              基址（Base Address）      | 预取位 | 类型    | 空间 |
|                                      |        |         | 指示 |
+--------------------------------------+--------+---------+------+

Bit 0 - 空间指示器（Space Indicator）:
        0 = 内存空间
        1 = I/O空间

Bit 2:1 - 内存类型（仅内存BAR）:
        00 = 32位
        10 = 64位
        01 = 保留（1MB以下，PCI遗留）
        11 = 保留

Bit 3 - 预取位（Prefetchable）:
        0 = 不可预取
        1 = 可预取
```

### 4.2.2 BAR编程过程

BAR的编程遵循特定的协议，软件通过"写入全1再读取"的方式来确定BAR的大小：

**步骤1：读取原始值**
```
软件读取BAR寄存器，保存原始值
```

**步骤2：写入全1**
```
软件向BAR写入0xFFFFFFFF（32位）或0xFFFFFFFFFFFFFFFF（64位）
```

**步骤3：读取大小掩码**
```
软件再次读取BAR，获取大小掩码
- 低位的1表示可编程位（决定大小）
- 高位返回设备地址需求
```

**步骤4：恢复原始值**
```
软件将原始值写回BAR
```

**步骤5：编程基址**
```
软件分配适当大小的对齐地址，写入BAR
```

**图4-3：BAR大小探测示例**

![IO地址路由](../images/图4-3.IO地址路由.jpg)

```
示例：设备需要64KB内存空间

初始状态:
BAR = 0x00000000 (未初始化)

步骤2 - 写入全1:
BAR = 0xFFFFFFFF

步骤3 - 读取掩码:
BAR = 0xFFFF0000  <-- 低16位为1，表示64KB对齐需求
                     高16位保留

计算大小:
可编程位 = 16位
大小 = 2^16 = 64KB

步骤5 - 编程基址:
软件分配0xF0000000
BAR = 0xF0000000  <-- 设备现在响应0xF0000000-0xF000FFFF
```

### 4.2.3 可预取与不可预取内存

**可预取内存（Prefetchable Memory）**
- 允许推测性读取和缓存
- 读取无副作用（幂等）
- 通常用于帧缓冲区等
- 示例：显卡显存

**不可预取内存（Non-Prefetchable）**
- 读取可能有副作用
- 不能进行推测性访问
- 用于设备控制寄存器
- 示例：控制/状态寄存器

**图4-4：内存类型对比**

![配置地址路由](../images/图4-4.配置地址路由.jpg)

```
可预取内存访问:
CPU → 读取地址A → 可缓存 → 后续读取可能从缓存获取
     ↓
    允许预取A+1, A+2等

不可预取内存访问:
CPU → 读取地址A → 不缓存 → 每次必须访问设备
     ↓
    不允许预取（可能有副作用）
```

### 4.2.4 Linux内核BAR处理

```c
// drivers/pci/probe.c

/**
 * __pci_read_base - 读取PCI BAR
 * @dev: PCI设备
 * @type: BAR类型
 * @res: 资源缓冲区
 * @pos: 配置空间中BAR位置
 *
 * 返回1表示64位BAR，0表示32位BAR
 */
int __pci_read_base(struct pci_dev *dev, enum pci_bar_type type,
            struct resource *res, unsigned int pos)
{
    u32 l = 0, sz = 0, mask;
    u64 l64, sz64, mask64;

    // 读取原始值
    pci_read_config_dword(dev, pos, &l);
    
    // 写入全1以探测大小
    pci_write_config_dword(dev, pos, l | mask);
    pci_read_config_dword(dev, pos, &sz);
    
    // 恢复原值
    pci_write_config_dword(dev, pos, l);

    // 计算BAR大小
    sz64 = pci_size(l64, sz64, mask64);
    
    // 处理64位BAR
    if (res->flags & IORESOURCE_MEM_64) {
        pci_read_config_dword(dev, pos + 4, &l);
        pci_write_config_dword(dev, pos + 4, ~0);
        pci_read_config_dword(dev, pos + 4, &sz);
        pci_write_config_dword(dev, pos + 4, l);
        
        l64 |= ((u64)l << 32);
        sz64 |= ((u64)sz << 32);
    }
}

/**
 * pci_size - 计算BAR大小
 */
static u64 pci_size(u64 base, u64 maxbase, u64 mask)
{
    u64 size = mask & maxbase;
    if (!size)
        return 0;
    
    // 找到最低有效位
    size = size & ~(size-1);
    
    return size;
}
```

---

## 4.3 桥接器地址窗口

### 4.3.1 Base/Limit寄存器

PCIe桥接器（包括交换机端口）使用Base/Limit寄存器来定义其下游的地址范围。这些寄存器允许桥接器：
- 确定哪些事务需要向下游转发
- 配置地址转换和路由

**Type 1配置头部中的地址窗口寄存器：**

```
偏移0x20: 内存基址/限制寄存器（I/O基址/限制高16位）
偏移0x24: 可预取内存基址/限制寄存器
偏移0x28: 可预取内存基址高32位
偏移0x2C: 可预取内存限制高32位
偏移0x30: I/O基址/限制高16位
```

**图4-5：桥接器内存窗口**

![消息路由](../images/图4-5.消息路由.jpg)

```
上游端口                    桥接器                     下游总线
    |                          |                           |
    |  TLP (地址=X)            |                           |
    |------------------------->|                           |
    |                          | 检查X是否在Base/Limit内   |
    |                          |                           |
    |                          | 是: 转发到下游            |
    |                          | 否: 忽略                  |
    |                          |                           |
    |                          |------------------------->|

Base/Limit寄存器定义的范围:
+------------------+
|  内存窗口        |  <-- 内存基址/限制寄存器
|  (非可预取)      |
+------------------+
|  I/O窗口         |  <-- I/O基址/限制寄存器
+------------------+
|  可预取内存窗口  |  <-- 可预取内存基址/限制寄存器
+------------------+
```

### 4.3.2 地址窗口编程

桥接器的地址窗口通常在枚举过程中由系统软件配置：

1. **扫描下游设备**：发现所有设备的BAR
2. **计算总需求**：累加所有BAR的大小
3. **分配地址范围**：在系统地址空间中分配连续区域
4. **编程Base/Limit**：将范围写入桥接器寄存器

**图4-6：地址窗口分配示例**

```
系统地址空间:
0x00000000 --------------------------------------------------
          |                                                |
          |   其他系统内存                                  |
          |                                                |
0xE0000000 --------------------------------------------------
          |                                                |
          |   桥接器0窗口 (64MB)                           |
          |   Base=0xE0000000, Limit=0xE3FFFFFF            |
          |   +----------------------------------------+   |
          |   |  设备0 BAR0: 0xE0000000 (16MB)         |   |
          |   |  设备1 BAR0: 0xE1000000 (32MB)         |   |
          |   |  设备2 BAR0: 0xE3000000 (16MB)         |   |
          |   +----------------------------------------+   |
          |                                                |
0xE4000000 --------------------------------------------------
          |                                                |
          |   桥接器1窗口 (128MB)                          |
          |   Base=0xE4000000, Limit=0xEBFFFFFF            |
          |                                                |
0xEC000000 --------------------------------------------------
```

---

## 4.4 事务类型

### 4.4.1 发布式与非发布式事务

PCIe定义了两类事务：

**发布式事务（Posted Transactions）**
- 请求者发送TLP后不等待完成
- 提高总线效率
- 包括：内存写入、消息

**非发布式事务（Non-Posted Transactions）**
- 请求者必须等待完成TLP
- 包括：内存读取、I/O读写、配置读写

**图4-7：事务类型对比**

```
发布式写入:
请求者                    完成者
   |---- MWr TLP -------->|
   |                      |
   |  立即继续            |
   |  无需等待            |
   v                      v

非发布式读取:
请求者                    完成者
   |---- MRd TLP -------->|
   |                      |
   |<--- CplD TLP --------|
   |                      |
   v  必须等待完成        v
```

### 4.4.2 事务排序规则

PCIe继承了PCI的事务排序模型，确保数据一致性：

**排序规则（Strict Ordering）**
- 同一路径上的写入按顺序完成
- 读取操作按顺序完成
- 写入后读取需要等待写入完成

**宽松排序（Relaxed Ordering）**
- 允许某些事务乱序完成
- 需要软件确保正确性
- 提高性能但增加复杂性

**图4-8：事务排序示例**

```
严格排序:
请求者                    完成者
   |---- MWr A ---------->|
   |---- MWr B ---------->|
   |                      |
   |  B必须在A之后完成    |
   |                      |
   v                      v

宽松排序:
请求者                    完成者
   |---- MWr A ---------->|
   |---- MWr B ---------->|
   |                      |
   |  B可以在A之前完成    |
   |  (如果设置了RO位)    |
   |                      |
   v                      v

发送器                  接收器
   |                      |
   |---- TLP ------------>|
   |    (消耗信用)        |
   |                      |
   |<--- FC DLLP ---------|
   |    (信用更新)        |
   |                      |
   |---- TLP ------------>|
   |    (有可用信用)      |
   |                      |
   v                      v

信用计数器:
初始: 信用 = 10
发送TLP: 信用 = 9
发送TLP: 信用 = 8
...
接收FC DLLP (+2): 信用 = 10
c
// drivers/pci/bus.c

/**
 * pci_bus_alloc_resource - 从父总线分配资源
 * @bus: PCI总线
 * @res: 要分配的资源
 * @size: 资源大小
 * @align: 对齐要求
 * @min: 最小地址
 * @type_mask: 资源类型标志
 */
int pci_bus_alloc_resource(struct pci_bus *bus, struct resource *res,
        resource_size_t size, resource_size_t align,
        resource_size_t min, unsigned long type_mask,
        resource_size_t (*alignf)(void *,
                      const struct resource *,
                      resource_size_t,
                      resource_size_t),
        void *alignf_data)
{
    // 尝试从64位区域分配
    if (res->flags & IORESOURCE_MEM_64) {
        rc = pci_bus_alloc_from_region(bus, res, size, align, min,
                       type_mask, alignf, alignf_data,
                       &pci_high);
        if (rc == 0)
            return 0;
    }
    
    // 回退到32位区域
    return pci_bus_alloc_from_region(bus, res, size, align, min,
                     type_mask, alignf, alignf_data,
                     &pci_32_bit);
}
```

### 4.7.2 桥窗口配置

```c
// drivers/pci/setup-bus.c

static void pci_setup_bridge_io(struct pci_bus *bus)
{
    struct pci_dev *bridge = bus->self;
    struct resource *res;
    
    // 配置I/O窗口
    res = bus->resource[0];  // I/O基址
    if (res) {
        pcibios_io_base = res->start;
        pci_write_config_dword(bridge, PCI_IO_BASE,
            ((res->start >> 8) & 0xF0) | ((res->end >> 8) & 0xF000));
    }
}

static void pci_setup_bridge_mmio(struct pci_bus *bus)
{
    struct pci_dev *bridge = bus->self;
    struct resource *res;
    
    // 配置内存窗口
    res = bus->resource[1];  // 内存基址
    if (res) {
        pci_write_config_word(bridge, PCI_MEMORY_BASE,
            (res->start >> 16) & 0xFFF0);
        pci_write_config_word(bridge, PCI_MEMORY_LIMIT,
            (res->end >> 16) & 0xFFF0);
    }
}
c
// 来自 drivers/pci/controller/pcie-phytium-register.h
#define PHYTIUM_PCI_BAR_0           0xe4
#define PHYTIUM_PCI_BAR(bar_num)    (0xe4 + bar_num * 4)
#define  BAR_IO_TYPE                (1 << 0)   // IO空间（对应4.2节）
#define  BAR_MEM_TYPE               (0 << 0)   // 存储器空间
#define  BAR_MEM_64BIT              (1 << 2)   // 64-bit BAR
#define  BAR_MEM_PREFETCHABLE       (1 << 3)   // 可预取
```

**与书中图4-2的对应**：

书中BAR格式：
```
Bit 0: 0=Memory, 1=IO
Bit 1: Reserved (0)
Bit 2: 0=32-bit, 1=64-bit
Bit 3: 0=Non-prefetchable, 1=Prefetchable
Bits 4-31: Base Address
```

飞腾宏定义完全对应：
- `BAR_IO_TYPE` (bit 0) ←→ 书中IO Space指示
- `BAR_MEM_64BIT` (bit 2) ←→ 书中64-bit指示
- `BAR_MEM_PREFETCHABLE` (bit 3) ←→ 书中Prefetchable指示

#### BAR设置代码实现

```c
// 来自 drivers/pci/controller/pcie-phytium-ep.c
static int phytium_pcie_ep_set_bar(struct pci_epc *epc, u8 fn,
                                   struct pci_epf_bar *epf_bar)
{
    struct phytium_pcie_ep *priv = epc_get_drvdata(epc);
    u64 sz = 0, sz_mask, atr_size;
    int flags = epf_bar->flags;
    u32 setting, trsl_param;
    enum pci_barno barno = epf_bar->barno;
    
    // 判断BAR类型（对应4.2节BAR类型）
    if ((flags & PCI_BASE_ADDRESS_SPACE) == PCI_BASE_ADDRESS_SPACE_IO) {
        // IO空间BAR
        setting = BAR_IO_TYPE;
        sz = max_t(size_t, epf_bar->size, BAR_IO_MIN_APERTURE);
        sz = 1 << fls64(sz - 1);
        sz_mask = ~(sz - 1);
        setting |= sz_mask;
        trsl_param = TRSL_ID_IO;
    } else {
        // 存储器空间BAR
        setting = BAR_MEM_TYPE;
        sz = max_t(size_t, epf_bar->size, BAR_MEM_MIN_APERTURE);
        sz = 1 << fls64(sz - 1);
        sz_mask = ~(sz - 1);
        setting |= lower_32_bits(sz_mask);
        
        // 64-bit BAR（对应4.2节64-bit BAR）
        if (flags & PCI_BASE_ADDRESS_MEM_TYPE_64)
            setting |= BAR_MEM_64BIT;
        
        // 可预取BAR（对应4.2节Prefetchable）
        if (flags & PCI_BASE_ADDRESS_MEM_PREFETCH)
            setting |= BAR_MEM_PREFETCHABLE;
        
        trsl_param = TRSL_ID_MASTER;
    }
    
    // 写入BAR寄存器
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_BAR(barno), setting);
    
    // 64-bit BAR的高32位
    if (flags & PCI_BASE_ADDRESS_MEM_TYPE_64)
        phytium_pcie_writel(priv, fn, PHYTIUM_PCI_BAR(barno + 1),
                            upper_32_bits(sz_mask));
    
    // 配置地址转换（ATR）
    atr_size = fls64(sz - 1) - 1;
    src_addr0 = ATR_IMPL | ((atr_size & ATR_SIZE_MASK) << ATR_SIZE_SHIFT);
    trsl_addr0 = (lower_32_bits(epf_bar->phys_addr) & TRSL_ADDR_32_12_MASK);
    trsl_addr1 = upper_32_bits(epf_bar->phys_addr);
    
    // 写入地址转换寄存器
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_WIN0_SRC_ADDR0(barno), src_addr0);
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_WIN0_TRSL_ADDR0(barno), trsl_addr0);
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_WIN0_TRSL_ADDR1(barno), trsl_addr1);
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_WIN0_TRSL_PARAM(barno), trsl_param);
    
    return 0;
}
```

#### 与书中内容的对应

1. **BAR类型判断**（对应4.2节）：
   - 书中介绍的IO/Memory类型判断在代码中通过`PCI_BASE_ADDRESS_SPACE`标志检查
   - 64-bit和Prefetchable标志检查与书中描述一致

2. **地址转换**（对应4.6节地址路由）：
   - 飞腾实现ATR（Address Translation）机制
   - `PHYTIUM_PCI_WIN0_SRC_ADDR`对应PCIe地址
   - `PHYTIUM_PCI_WIN0_TRSL_ADDR`对应本地物理地址
   - 实现了书中介绍的地址路由功能

3. **BAR清除**（对应4.2节BAR响应）：
   ```c
   static void phytium_pcie_ep_clear_bar(struct pci_epc *epc, u8 fn,
                                         struct pci_epf_bar *epf_bar)
   {
       // 清除BAR寄存器（对应书中BAR响应0）
       phytium_pcie_writel(priv, fn, PHYTIUM_PCI_BAR(barno), 0);
       if (flags & PCI_BASE_ADDRESS_MEM_TYPE_64)
           phytium_pcie_writel(priv, fn, PHYTIUM_PCI_BAR(barno + 1), 0);
       
       // 清除地址转换
       phytium_pcie_writel(priv, fn, PHYTIUM_PCI_WIN0_SRC_ADDR0(barno), 0);
       // ...
   }
   ```

### 4.8.2 地址窗口配置

飞腾PCIe EP支持多个地址转换窗口：

```c
// 地址窗口寄存器定义
#define PHYTIUM_PCI_WIN0_BASE           0x600
#define PHYTIUM_PCI_WIN0_SRC_ADDR0(table)   (PHYTIUM_PCI_WIN0_BASE + 0X20 * table + 0x0)
#define PHYTIUM_PCI_WIN0_TRSL_ADDR0(table)  (PHYTIUM_PCI_WIN0_BASE + 0X20 * table + 0x8)
#define PHYTIUM_PCI_WIN0_TRSL_PARAM(table)  (PHYTIUM_PCI_WIN0_BASE + 0X20 * table + 0x10)
```

**与书中4.4节的对应**：
- 书中介绍的Base/Limit寄存器在飞腾实现中对应`SRC_ADDR`和`TRSL_ADDR`寄存器
- 支持多个窗口（由`table`参数索引），对应书中介绍的多个地址范围

### 4.8.3 实际应用建议

**BAR调试技巧**：
1. 使用`cat /sys/bus/pci/devices/.../resource`查看BAR分配
2. 检查BAR值是否与书中4.2节描述的格式一致
3. 验证地址转换是否正确配置

**性能优化**：
- 对于大容量存储器，使用64-bit BAR（对应4.2节64-bit BAR）
- 对于只读存储器，设置Prefetchable位（对应4.2节Prefetchable）
- 合理配置地址窗口大小，避免地址空间浪费

---

**翻译完成**

*本翻译基于MindShare《PCI Express Technology 3.0》第4章内容，并补充了飞腾平台BAR实现参考，保留所有技术术语的准确性。*

---

## 本章图片附录

以下是本章相关的原文图片：

### 图4-1.地址空间.png

![图4-1.地址空间.png](../images/图4-1.地址空间.png)

### 图4-2.内存地址路由.jpg

![图4-2.内存地址路由.jpg](../images/图4-2.内存地址路由.jpg)

### 图4-3.IO地址路由.jpg

![图4-3.IO地址路由.jpg](../images/图4-3.IO地址路由.jpg)

### 图4-4.配置地址路由.jpg

![图4-4.配置地址路由.jpg](../images/图4-4.配置地址路由.jpg)

### 图4-5.消息路由.jpg

![图4-5.消息路由.jpg](../images/图4-5.消息路由.jpg)

