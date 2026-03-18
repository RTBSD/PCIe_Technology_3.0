# 第15章 错误检测与处理 (Error Detection and Handling)

## 目录
- [背景](#背景)
- [PCIe错误定义](#pcie错误定义)
- [PCIe错误报告](#pcie错误报告)
  - [基线错误报告](#基线错误报告)
  - [高级错误报告 (AER)](#高级错误报告-aer)
- [错误分类](#错误分类)
  - [可纠正错误](#可纠正错误)
  - [不可纠正错误](#不可纠正错误)
    - [非致命不可纠正错误](#非致命不可纠正错误)
    - [致命不可纠正错误](#致命不可纠正错误)
- [PCIe错误检查机制](#pcie错误检查机制)
  - [CRC](#crc)
  - [各层错误检查](#各层错误检查)
    - [物理层错误](#物理层错误)
    - [数据链路层错误](#数据链路层错误)
    - [事务层错误](#事务层错误)
- [错误污染](#错误污染)
- [PCI Express错误来源](#pci-express错误来源)
  - [ECRC生成与检查](#ecrc生成与检查)
    - [TLP摘要](#tlp摘要)
    - [ECRC机制中不包含的变体位](#ecrc机制中不包含的变体位)
  - [数据污染](#数据污染)
  - [拆分事务错误](#拆分事务错误)
    - [不支持的请求 (UR) 状态](#不支持的请求-ur-状态)
    - [完成者中止 (CA) 状态](#完成者中止-ca-状态)
    - [意外完成](#意外完成)
    - [完成超时](#完成超时)
  - [链路流量控制相关错误](#链路流量控制相关错误)
  - [畸形TLP](#畸形tlp)
  - [内部错误](#内部错误)
- [错误如何报告](#错误如何报告)
  - [简介](#简介)
  - [错误消息](#错误消息)
    - [咨询性非致命错误](#咨询性非致命错误)
    - [咨询性非致命错误情况](#咨询性非致命错误情况)
- [基线错误检测与处理](#基线错误检测与处理)
  - [PCI兼容错误报告机制](#pci兼容错误报告机制)
    - [概述](#概述)
    - [传统命令和状态寄存器](#传统命令和状态寄存器)
  - [基线错误处理](#基线错误处理)
    - [启用/禁用错误报告](#启用禁用错误报告)
    - [设备控制寄存器](#设备控制寄存器)
    - [设备状态寄存器](#设备状态寄存器)
    - [根复合体对错误消息的响应](#根复合体对错误消息的响应)
  - [链路错误](#链路错误)
- [高级错误报告 (AER)](#高级错误报告-aer-1)
  - [高级错误能力与控制](#高级错误能力与控制)
  - [处理粘性位](#处理粘性位)
  - [高级可纠正错误处理](#高级可纠正错误处理)
    - [高级可纠正错误状态](#高级可纠正错误状态)
    - [高级可纠正错误屏蔽](#高级可纠正错误屏蔽)
  - [高级不可纠正错误处理](#高级不可纠正错误处理)
    - [高级不可纠正错误状态](#高级不可纠正错误状态)
    - [选择不可纠正错误严重性](#选择不可纠正错误严重性)
    - [不可纠正错误屏蔽](#不可纠正错误屏蔽)
  - [头部日志记录](#头部日志记录)
  - [根复合体错误跟踪与报告](#根复合体错误跟踪与报告)
    - [根复合体错误状态寄存器](#根复合体错误状态寄存器)
    - [高级源ID寄存器](#高级源id寄存器)
    - [根错误命令寄存器](#根错误命令寄存器)
- [错误日志记录与报告总结](#错误日志记录与报告总结)
- [软件错误调查示例流程](#软件错误调查示例流程)

---

## 背景

PCIe通过使用传统配置寄存器中的错误状态位来保持与这些传统机制的向后兼容性，以记录PCIe中与PCI类似的错误事件。这使得传统软件能够以它理解的方式查看PCIe错误事件，并允许它与PCIe硬件一起运行。

PCI-X 2.0使用源同步时钟来实现更快的数据速率（高达4GB/s）。该总线针对高端企业系统，因为它对消费者机器来说通常太贵了。由于这些高性能系统还需要高可用性，规范编写者选择通过添加纠错码（ECC）支持来改进错误处理。ECC允许更强大的错误检测，并能够即时纠正单比特错误。ECC对于最小化传输错误的影响非常有帮助。

PCIe通过使用传统配置寄存器中的错误状态位来保持与这些传统机制的向后兼容性，以记录PCIe中与PCI类似的错误事件。这使得传统软件能够以它理解的方式查看PCIe错误事件，并允许它与PCIe硬件一起运行。

---

## PCIe错误定义

规范使用四个关于错误的一般术语，定义如下：

1. **错误检测 (Error Detection)** — 确定存在错误的过程。错误由代理发现，原因是本地问题（如接收到坏包）或因为它从另一设备接收到信号错误的包（如被污染的数据包）。

2. **错误日志记录 (Error Logging)** — 根据检测到的错误在架构寄存器中设置适当的位，以帮助错误处理软件。

3. **错误报告 (Error Reporting)** — 通知系统存在错误条件。这可以采取错误消息传递到根复合体（Root Complex）的形式，假设设备被启用以发送错误消息。根复合体在接收到错误消息时，可以向系统发送中断。

4. **错误信号 (Error Signaling)** — 一个代理通过发送错误消息、发送带有UR（不支持的请求）或CA（完成者中止）状态的完成包，或污染TLP（也称为错误转发）来通知另一个代理错误条件的过程。

---

## PCIe错误报告

PCIe定义了两个错误报告级别。第一个是**基线能力 (Baseline capability)**，所有设备都必须支持。这包括对传统错误报告的支持以及对报告PCIe错误的基本支持。第二个是可选的**高级错误报告能力 (Advanced Error Reporting Capability)**，它添加了一组新的配置寄存器，并跟踪更多关于发生了哪些错误、它们的严重性如何的详细信息，在某些情况下，甚至可以记录导致错误的包的信息。

### 基线错误报告

所有设备都需要两组配置寄存器来支持基线错误报告：

- **PCI兼容寄存器 (PCI-compatible Registers)** — 这些是与PCI使用的相同寄存器，为现有的PCI兼容软件提供向后兼容性。为了实现这一点，PCIe错误被映射到PCI兼容错误，使它们对旧软件可见。

- **PCI Express能力寄存器 (PCI Express Capability Registers)** — 这些寄存器只对了解PCIe的新软件有用，但它们为PCIe软件提供了更多的错误信息。

### 高级错误报告 (AER)

这种可选的错误报告机制包括一组新的专用配置寄存器，为错误处理软件提供更多信息和诊断能力。AER寄存器映射到扩展配置空间中，提供关于任何错误性质的更多信息。

---

## 错误分类

错误分为两大类，基于硬件是否能够修复问题：**可纠正 (Correctable)** 和 **不可纠正 (Uncorrectable)**。不可纠正类别进一步细分为基于软件是否能够修复问题：**非致命 (Non-fatal)** 和 **致命 (Fatal)**。

| 错误类别 | 描述 |
|---------|------|
| 可纠正错误 (Correctable errors) | 由硬件自动处理 |
| 不可纠正错误 (Uncorrectable errors) | 需要软件干预 |
| ├─ 非致命 (Non-fatal) | 由设备特定软件处理；链路仍可运行，可能恢复而不丢失数据 |
| └─ 致命 (Fatal) | 由系统软件处理；链路或设备无法正常工作，恢复而不丢失数据不太可能 |

基于这些类别，错误处理软件可以划分为单独的处理器来执行所需的操作。这些操作可能范围从简单地监控可纠正错误的频率到在发生致命错误时重置整个系统。无论错误类型如何，软件都可以安排系统被通知所有错误，以允许跟踪和记录它们。

### 可纠正错误

可纠正错误，根据定义，在硬件中自动纠正。它们可能通过增加延迟和消耗带宽来影响性能，但如果一切顺利，恢复是自动且快速的，因为它不依赖于软件干预，并且在过程中不会丢失信息。这些错误不需要报告给软件，但这样做可以让软件跟踪错误趋势，这可能表明某些设备显示出即将故障的迹象。

### 不可纠正错误

无法在硬件中自动纠正的错误称为不可纠正错误，这些错误在严重性上要么是非致命的，要么是致命的。

#### 非致命不可纠正错误

非致命错误表明信息已丢失，但原因可能是链路或设备完整性以外的问题。某个地方的数据包失败了，但链路继续正常运行，其他数据包不受影响。由于链路仍在工作，丢失信息的恢复可能是可能的，但这将依赖于实现特定的软件来处理它。这种错误类型的一个例子是**完成超时 (Completion timeout)**，即发送了请求但在允许的时间内没有返回完成包。某个地方有问题，但它可能是交换机内的随机比特错误导致完成包路由不正确这样简单的问题。这种情况的恢复尝试可以像重新发出请求一样简单。

#### 致命不可纠正错误

致命错误表明链路或设备发生了操作故障，导致数据丢失不太可能恢复。对于这些情况，重置至少失败的链路或设备可能是任何恢复过程中的第一步，因为它显然由于某种原因无法正常工作。规范还允许实现特定的方法，其中软件可能试图限制故障的影响，但它没有定义应该采取的任何特定操作。这种错误类型的一个例子是**接收缓冲区溢出 (receiver buffer overflow)**，在这种情况下，信息已经丢失，因为流量控制跟踪计数器彼此失去同步。由于没有机制来修复这个问题，通常需要重置此链路。

---

## PCIe错误检查机制

PCIe错误检查的范围集中在与链路和数据包传递相关的错误上。与链路传输无关的错误不会通过PCIe错误处理机制报告，而需要专有方法来报告它们，例如设备特定的中断。

接口的每一层都包括错误检查能力：

### CRC

在深入讨论与层相关的错误处理之前，首先讨论**CRC (Cyclic Redundancy Check，循环冗余校验)** 的概念会很有帮助，因为它是PCIe错误检查的组成部分。

CRC代码由发送器基于数据包内容计算，并将其添加到数据包中进行传输。CRC名称来源于这样一个事实：这个检查代码（从要检查错误的数据包计算得出）是冗余的（不向数据包添加信息），并且来自循环码。

虽然CRC不能提供足够的信息像ECC（纠错码）那样进行自动纠错，但它确实提供了强大的错误检测能力。CRC也常用于串行传输，因为它们擅长检测一串错误的比特。

CRC在PCIe中有两种不同的使用场景：

1. **LCRC (Link CRC，链路CRC)** — 在数据链路层为每个跨链路传输的TLP生成和检查。它旨在检测链路上的传输错误。

2. **ECRC (End-to-end CRC，端到端CRC)** — 在发送器的事务层生成，在数据包的最终目标的事务层检查。这旨在检测否则可能是静默的错误，例如当TLP通过交换机（Switch）等中间代理时。

### 各层错误检查

在接收器处，传入数据包的不同方面在不同的层进行检查。某些错误检查被列为可选。对于这些情况，如果发生错误但设计人员选择不实现该形式的检查，则将不会检测到它。

#### 物理层错误

到达接收器的数据包首先到达物理层。在这个级别必须检查一些东西，其他东西可以选择性检查。

物理层错误，也称为**接收器错误 (Receiver Errors)** 或 **链路错误 (Link Errors)**，包括以下情况：

| 错误类型 | 检查要求 |
|---------|---------|
| 使用8b/10b时的解码违规 | 必需检查 |
| 帧违规 | 8b/10b可选，128b/130b必需 |
| 弹性缓冲区错误 | 可选检查 |
| 符号锁定丢失或通道去偏斜 | 可选检查 |

如果在检测到接收器错误时TLP正在进行中，则将其丢弃。为了解决错误，向数据链路层发送信号以发送NAK（如果尚未挂起）。

#### 数据链路层错误

在物理层之后，传入数据包接下来进入数据链路层，在那里检查几个可能的问题：

- TLP的LCRC失败
- TLP的序列号违规
- DLLP的16位CRC失败
- 链路层协议错误

与物理层一样，如果在看到错误时TLP正在进行中，则丢弃TLP，并安排NAK（如果尚未挂起）。

在发送器端也有一些数据链路层错误需要注意，包括：
- REPLAY_TIMER过期
- REPLAY_NUM计数器溢出

超时通过重放缓冲区（Replay Buffer）的内容来处理，并递增REPLAY_NUM计数器。当到达发送器的ACK或NAK表明已经取得进展时（意味着它导致从重放缓冲区清除一个或多个TLP），定时器和计数器被重置。但是，如果没有足够快地收到Ack或Nak，则会看到超时条件，这将导致重放。

#### 事务层错误

最后，如果传入的TLP通过了物理层和数据链路层的所有检查，它们最终将到达事务层，在那里检查：

| 错误检查 | 要求 |
|---------|------|
| ECRC失败 | 可选 |
| 畸形TLP（数据包格式错误） | 必需 |
| 不支持的请求 | 必需 |
| 完成者中止 | 必需 |
| 完成超时 | 必需 |
| 意外完成 | 必需 |

---

## Linux内核实现说明

### AER驱动架构

Linux内核通过`drivers/pci/pcie/aer.c`实现PCIe高级错误报告支持：

```c
// AER错误源结构
struct aer_err_source {
    unsigned int status;
    unsigned int id;
};

// 错误统计
struct aer_stats {
    u64 dev_cor_errs[AER_MAX_TYPEOF_COR_ERRS];      // 16种可纠正错误
    u64 dev_fatal_errs[AER_MAX_TYPEOF_UNCOR_ERRS];  // 27种致命错误
    u64 dev_nonfatal_errs[AER_MAX_TYPEOF_UNCOR_ERRS]; // 27种非致命错误
};
```

### 关键接口

1. **启用/禁用错误报告**
   ```c
   int pci_enable_pcie_error_reporting(struct pci_dev *dev);
   int pci_disable_pcie_error_reporting(struct pci_dev *dev);
   ```

2. **清除错误状态**
   ```c
   int pci_aer_clear_nonfatal_status(struct pci_dev *dev);
   void pci_aer_clear_fatal_status(struct pci_dev *dev);
   ```

3. **ECRC控制**
   - 内核参数: `ecrc_policy=bios|off|on`
   - 通过`enable_ecrc_checking()`/`disable_ecrc_checking()`控制

### 错误处理流程

1. 根端口检测到错误 → 触发AER中断
2. `aer_isr()`收集错误源信息
3. 调度工作队列`aer_isr_work_func()`处理
4. 根据错误类型执行恢复操作
|---

## 15.12 Linux内核实现参考（平台特定补充）

### 15.12.1 Linux AER驱动实现

Linux内核的AER（Advanced Error Reporting）驱动实现了书中第15章介绍的高级错误报告机制。

#### AER寄存器定义

```c
// 来自 drivers/pci/pcie/aer.c
#define PCI_ERR_UNCOR_STATUS            0x04    // 不可纠正错误状态
#define PCI_ERR_UNCOR_MASK              0x08    // 不可纠正错误掩码
#define PCI_ERR_UNCOR_SEVER             0x0c    // 不可纠正错误严重程度
#define PCI_ERR_COR_STATUS              0x10    // 可纠正错误状态
#define PCI_ERR_COR_MASK                0x14    // 可纠正错误掩码
#define PCI_ERR_CAP                     0x18    // 错误能力

// 不可纠正错误位定义（对应书中15.5节）
#define PCI_ERR_UNC_DLP                 0x00000010      // 数据链路层协议错误
#define PCI_ERR_UNC_SURPDN              0x00000020      // 意外下行
#define PCI_ERR_UNC_POISON_TLP          0x00001000      // 毒化TLP
#define PCI_ERR_UNC_FCP                 0x00002000      // 流量控制协议错误
#define PCI_ERR_UNC_COMP_TIME           0x00004000      // 完成超时
#define PCI_ERR_UNC_COMP_ABORT          0x00008000      // 完成中止
#define PCI_ERR_UNC_UNX_COMP            0x00010000      // 意外完成
#define PCI_ERR_UNC_RX_OVER             0x00020000      // 接收溢出
#define PCI_ERR_UNC_MALF_TLP            0x00040000      // 畸形TLP
#define PCI_ERR_UNC_ECRC                0x00080000      // ECRC错误
#define PCI_ERR_UNC_UNSUP               0x00100000      // 不支持请求

// 可纠正错误位定义（对应书中15.5节）
#define PCI_ERR_COR_RCVR                0x00000001      // 接收器错误
#define PCI_ERR_COR_BAD_TLP             0x00000040      // 坏TLP
#define PCI_ERR_COR_BAD_DLLP            0x00000080      // 坏DLLP
#define PCI_ERR_COR_REP_ROLL            0x00000100      // 重放缓存溢出
#define PCI_ERR_COR_REP_TIMER           0x00001000      // 重放定时器超时
c
// AER错误信息结构（对应书中15.10节）
struct aer_err_info {
    struct pci_dev *dev;            // 错误设备
    int error_weight;               // 错误权重
    u32 status;                     // 错误状态
    u32 mask;                       // 错误掩码
    u32 severity;                   // 严重程度
    u32 uncor_status;               // 不可纠正错误状态
    u32 cor_status;                 // 可纠正错误状态
    u32 header_log[4];              // TLP头日志（对应书中15.10节）
    u32 tlp_header_valid;           // TLP头有效
};

// AER端口服务数据
struct aer_rpc {
    struct pci_dev *rpd;            // 根端口设备
    struct work_struct aer_work;    // 错误处理工作队列
    int aer_irq;                    // AER中断号
};
```

#### AER中断处理流程

```c
// AER中断服务程序（对应书中15.10节错误处理流程）
static irqreturn_t aer_isr(int irq, void *context)
{
    struct pcie_device *adev = context;
    struct aer_rpc *rpc = get_service_data(adev);
    
    // 1. 读取根端口错误状态（对应书中15.10节）
    // 2. 收集错误源信息
    // 3. 调度工作队列处理
    schedule_work(&rpc->aer_work);
    
    return IRQ_HANDLED;
}

// AER错误处理工作函数
static void aer_isr_work_func(struct work_struct *work)
{
    struct aer_rpc *rpc = container_of(work, struct aer_rpc, aer_work);
    struct aer_err_info info;
    
    // 1. 获取错误信息
    aer_get_error_info(rpc, &info);
    
    // 2. 根据错误类型处理（对应书中15.4节错误分类）
    if (info.severity == AER_FATAL) {
        // 致命错误处理
        aer_fatal_error(&info);
    } else if (info.severity == AER_NONFATAL) {
        // 非致命错误处理
        aer_nonfatal_error(&info);
    } else {
        // 可纠正错误处理
        aer_correctable_error(&info);
    }
    
    // 3. 清除错误状态
    aer_clear_error(&info);
}
```

**与书中15.10节的对应**：

书中AER处理流程：
1. 错误检测 → 2. 错误日志 → 3. 错误报告 → 4. 错误恢复

Linux实现：
1. `aer_isr()` - 中断检测
2. `aer_get_error_info()` - 收集错误日志（包括TLP头）
3. `aer_isr_work_func()` - 错误报告和处理
4. `aer_fatal/nonfatal/correctable_error()` - 错误恢复

#### 错误恢复机制

```c
// 致命错误恢复（对应书中15.10节）
static void aer_fatal_error(struct aer_err_info *info)
{
    struct pci_dev *dev = info->dev;
    
    // 1. 停止设备
    pci_stop_dev(dev);
    
    // 2. 执行链路复位
    pcie_reset_link(dev);
    
    // 3. 重新初始化设备
    pci_restore_state(dev);
    pci_set_master(dev);
    
    // 4. 重新启动设备
    pci_start_dev(dev);
}

// 非致命错误恢复
static void aer_nonfatal_error(struct aer_err_info *info)
{
    // 清除错误状态，继续运行
    aer_clear_error(info);
}

// 可纠正错误处理
static void aer_correctable_error(struct aer_err_info *info)
{
    // 仅清除错误状态
    pci_write_config_dword(info->dev, PCI_ERR_COR_STATUS, info->cor_status);
}
```

### 15.12.2 飞腾平台AER支持

飞腾PCIe控制器支持AER功能，与Linux AER驱动配合工作：

```c
// 飞腾AER相关寄存器（假设）
#define PHYTIUM_PCIE_AER_UNCOR_STATUS   0x104   // 不可纠正错误状态
#define PHYTIUM_PCIE_AER_COR_STATUS     0x110   // 可纠正错误状态
#define PHYTIUM_PCIE_AER_ROOT_STATUS    0x130   // 根端口状态
#define PHYTIUM_PCIE_AER_IRQ_EN         (1 << 0) // AER中断使能
```

**调试技巧**：
1. 查看AER错误统计：`cat /sys/bus/pci/devices/.../aer_*`
2. 启用AER日志：`echo 1 > /sys/bus/pci/devices/.../aer_verbosity`
3. 检查错误类型：`lspci -vv | grep -A 10 "Advanced Error Reporting"`

### 15.12.3 实际应用建议

**错误处理策略**：
- 可纠正错误：自动清除，记录日志
- 非致命错误：清除错误，尝试恢复
- 致命错误：链路复位，重新初始化

**性能优化**：
- 合理设置错误掩码，避免中断风暴
- 使用工作队列处理错误，避免中断上下文阻塞
- 定期检查错误统计，预防潜在问题

**故障排查**：
- 检查PCIe链路质量（信号完整性）
- 验证电源稳定性
- 检查设备散热情况

---

*翻译完成于: 2026-03-17*
*原文来源: MindShare PCI Express Technology 3.0, Chapter 15*
*平台补充: Linux AER实现参考*

---

## 本章图片附录

以下是本章相关的原文图片：

### 图15-1.错误分类.png

![图15-1.错误分类.png](../images/图15-1.错误分类.png)

### 图15-2.错误检测.jpg

![图15-2.错误检测.jpg](../images/图15-2.错误检测.jpg)

### 图15-3.错误报告.png

![图15-3.错误报告.png](../images/图15-3.错误报告.png)

