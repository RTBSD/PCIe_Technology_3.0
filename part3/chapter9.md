# 第9章 DLLP元素

## 内容来源标注规则

本翻译内容来源于《PCI Express Technology 3.0》第9章"DLLP Elements"（第307-316页）。

| 标记 | 含义 |
|------|------|
| `[原文:章节]` | 来自参考文档的具体章节 |

---

## 上一章回顾

上一章讨论了PCI Express拓扑中事务的排序要求。这些规则继承自PCI，生产者/消费者编程模型是其中的主要动机，因此本章描述了其机制。原始规则还考虑了必须避免的可能死锁条件，但不包括避免可能导致性能问题的任何手段。

## 本章内容

在本章中，我们描述了另一大类数据包——**数据链路层包（DLLP, Data Link Layer Packet）**。我们描述了DLLP数据包类型的用途、格式和定义，以及其相关字段的详细信息。DLLP用于支持**Ack/Nak协议**、**电源管理**、**流量控制机制**，甚至可以用于**厂商自定义用途**。

## 下一章预告

下一章描述了数据链路层的一个关键特性：一个自动的、基于硬件的机制，用于确保TLP在链路上的可靠传输。**Ack DLLP**确认TLP的正确接收，而**Nak DLLP**表示传输错误。我们将描述在没有检测到TLP或DLLP错误时的正常操作规则，以及与TLP和DLLP错误相关的错误恢复机制。

---

## 9.1 概述（General）

数据链路层可以被认为管理着底层链路协议。其主要职责是确保在设备之间传输的**TLP（事务层包）**的完整性，但它也在TLP流量控制、链路初始化和电源管理中发挥作用，并在其上的事务层和其下的物理层之间传递信息。

在执行这些任务时，数据链路层与其邻居交换称为**数据链路层包（DLLP）**的数据包。DLLP在每台设备的数据链路层之间进行通信。图9-1展示了在设备之间交换的DLLP。

*[原文:第307-308页]*

---

## 9.2 DLLP是本地流量（DLLPs Are Local Traffic）

DLLP具有简单的数据包格式，大小固定为**8字节**（包括帧字节）。与TLP不同，它们不携带目标或路由信息，因为它们仅用于最近邻通信，根本不进行路由。事务层也看不到它们，因为它们不是在该级别交换的信息的一部分。

*[原文:第308页]*

---

## 9.3 接收器对DLLP的处理（Receiver handling of DLLPs）

当接收到DLLP时，适用以下几条规则：

1. **立即处理**：它们在接收器处立即处理。换句话说，它们的流量不能像TLP那样被控制（DLLP不受流量控制约束）。

2. **错误检查**：首先由物理层检查错误，然后由数据链路层检查。通过计算CRC应该是什么并将其与接收到的值进行比较来检查数据包中包含的**16位CRC**。未通过此检查的DLLP将被丢弃。链路如何从这种错误中恢复？DLLP仍会定期到达，下一个成功接收的该类型DLLP将更新缺失的信息。

3. **无确认协议**：与TLP不同，DLLP没有确认协议。相反，规范定义了**超时机制**以便于从失败的DLLP中恢复。

4. **分类处理**：如果没有错误，则确定DLLP类型并传递给适当的内部逻辑进行管理：
   - **Ack/Nak** - TLP状态通知
   - **流量控制** - 可用缓冲区空间通知
   - **电源管理**设置
   - **厂商特定信息**

*[原文:第309页]*

---

## 9.4 发送DLLP（Sending DLLPs）

### 9.4.1 概述（General）

这些数据包起源于数据链路层并传递给物理层。如果使用**8b/10b编码**（Gen1和Gen2模式），在发送数据包之前，物理层会在DLLP的两端添加帧符号。在**Gen3模式**下，会在DLLP前端添加一个2字节的**SDP令牌**，但不会在DLLP末尾添加END。

图9-2展示了一个通用（Gen1/Gen2）DLLP在传输中的状态，显示了帧符号和数据包的一般内容。

*[原文:第309-310页]*

---

### 9.4.2 DLLP数据包大小固定为8字节

数据链路层包在**8b/10b**和**128b/130b**编码下始终为**8字节**长，由以下部分组成：

1. **1个DW（双字）核心（4字节）**：包含1字节的DLLP类型字段和3个额外的属性字节。属性因DLLP类型而异。

2. **2字节CRC值**：根据DLLP的核心内容计算。需要指出的是，此CRC与添加到TLP的**LCRC（链路CRC）**不同。此CRC仅为**16位**，计算方式与TLP中的**32位LCRC**不同。此CRC附加到核心DLLP，然后将这6字节传递给物理层。

3. **8b/10b编码**：如果使用8b/10b编码，会在数据包的开头和结尾分别添加**DLLP起始（SDP, Start of DLLP）**控制符号和**良好结束（END, End Good）**控制符号。像往常一样，在传输之前，物理层将字节编码为10位符号进行传输。

4. **Gen3模式**：在Gen3模式下，当使用**128b/130b编码**时，会在数据包前端添加一个**2字节SDP令牌**以创建8字节数据包，并且没有END符号或令牌。

注意，DLLP从不携带数据有效载荷；所有信息都承载在数据包的核心4字节中。

*[原文:第310-311页]*

---

## 9.5 DLLP数据包类型（DLLP Packet Types）

定义了**四组DLLP**，分别处理**Ack/Nak**、**电源管理**和**流量控制**，以及一个**厂商特定版本**。其中一些有几个变体，表9-1总结了每个变体及其DLLP类型字段编码。

### 表9-1：DLLP类型

| DLLP类型 | 类型字段编码 | 用途 |
|----------|-------------|------|
| **Ack** (TLP确认) | 0000 0000b | TLP传输完整性 |
| **Nak** (TLP否定确认) | 0001 0000b | TLP传输完整性 |
| **PM_Enter_L1** | 0010 0000b | 电源管理 |
| **PM_Enter_L23** | 0010 0001b | 电源管理 |
| **PM_Active_State_Request_L1** | 0010 0011b | 电源管理 |
| **PM_Request_Ack** | 0010 0100b | 电源管理 |
| **Vendor Specific** (厂商特定) | 0011 0000b | 厂商定义 |
| **InitFC1-P** | 0100 0xxxb | TLP流量控制 (xxx = VC编号) |
| **InitFC1-NP** | 0101 0xxxb | TLP流量控制 |
| **InitFC1-Cpl** | 0110 0xxxb | TLP流量控制 |
| **InitFC2-P** | 1100 0xxxb | TLP流量控制 |
| **InitFC2-NP** | 1101 0xxxb | TLP流量控制 |
| **InitFC2-Cpl** | 1110 0xxxb | TLP流量控制 |
| **UpdateFC-P** | 1000 0xxxb | TLP流量控制 |
| **UpdateFC-NP** | 1001 0xxxb | TLP流量控制 |
| **UpdateFC-Cpl** | 1010 0xxxb | TLP流量控制 |
| **Reserved** (保留) | 其他 | 保留 |

*[原文:第311-312页]*

---

## 9.6 Ack/Nak DLLP格式（Ack/Nak DLLP Format）

设备用于**Ack（确认）**或**Nak（否定确认）**TLP接收的DLLP格式如图9-3所示，其字段在表9-2中描述。有关如何使用这些来处理Ack/Nak协议的更多讨论，请参阅第10章"Ack/Nak协议"（第317页）。

### 表9-2：Ack/Nak DLLP字段

| 字段名 | 头部字节/位 | DLLP功能 |
|--------|------------|----------|
| **DLLP类型** | Byte 0, [7:0] | 指示DLLP类型：<br>• 0000 0000b = Ack<br>• 0001 0000b = Nak |
| **AckNak_Seq_Num** | Byte 2, [3:0]<br>Byte 3, [7:0] | 如果接收到良好的TLP：<br>• 如果传入序列号 = NEXT_RCV_SEQ（与预期匹配），调度带有该编号的Ack DLLP。<br>• 如果传入序列号早于NEXT_RCV_SEQ计数（接收到重复的TLP），调度带有NEXT_RCV_SEQ - 1的Ack DLLP（实际上是最后一个良好TLP的编号）。<br><br>对于接收到有问题的TLP：<br>• 如果TLP有错误，或其序列号高于NEXT_RCV_SEQ，调度带有NEXT_RCV_SEQ - 1的Nak DLLP。 |
| **16位CRC** | Byte 4, [7:0]<br>Byte 5, [7:0] | 此16位CRC保护此DLLP的内容。计算基于Ack/Nak的字节0-3。 |

*[原文:第312-313页]*

---

## 9.7 电源管理DLLP格式（Power Management DLLP Format）

电源管理DLLP信息如图9-4所示，其字段在表9-3中描述。要了解有关在电源管理中使用这些数据包的更多信息，请参阅第16章"电源管理"（第703页）。

### 表9-3：电源管理DLLP字段

| 字段名 | 头部字节/位 | DLLP功能 |
|--------|------------|----------|
| **DLLP类型** | Byte 0, [7:0] | 指示DLLP类型。对于电源管理DLLP：<br>0010 0000b = PM_Enter_L1<br>0010 0001b = PM_Enter_L23<br>0010 0011b = PM_Active_State_Request_L1<br>0010 0100b = PM_Request_Ack |
| **16位CRC** | Byte 4, [7:0]<br>Byte 5, [7:0] | 用于保护DLLP内容的16位CRC。计算基于字节0-3，无论字段是否使用。 |

*[原文:第313-314页]*

---

## 9.8 流量控制DLLP格式（Flow Control DLLP Format）

与许多其他串行传输总线一样，PCIe通过使用**基于信用的流量控制方案**来提高传输效率。此主题在第6章"流量控制"（第215页）中有详细讨论。DLLP用于通信流量控制信用信息。各种不同的DLLP初始化流量控制信用。另一类**更新DLLP**用于在接收器缓冲区空间恢复时管理运行时信用管理。有两种**流量控制初始化DLLP**称为**InitFC1**和**InitFC2**，以及一种**流量控制更新DLLP**称为**UpdateFC**。

所有三种变体的数据包格式如图9-5所示，而表9-4描述了其中包含的字段。

### 表9-4：流量控制DLLP字段

| 字段名 | 头部字节/位 | DLLP功能 |
|--------|------------|----------|
| **DLLP类型** | Byte 0, [7:4] | 此代码指示FC DLLP的类型：<br>0100b = InitFC1-P (Posted请求)<br>0101b = InitFC1-NP (Non-Posted请求)<br>0110b = InitFC1-Cpl (完成)<br>1100b = InitFC2-P (Posted请求)<br>1101b = InitFC2-NP (Non-Posted请求)<br>1110b = InitFC2-Cpl (完成)<br>1000b = UpdateFC-P (Posted请求)<br>1001b = UpdateFC-NP (Non-Posted请求)<br>1010b = UpdateFC-Cpl (完成) |
| | Byte 0, [3] | 必须为0b，作为流量控制编码的一部分。 |
| | Byte 0, [2:0] | **VC ID**。指示要用这些信用更新的**虚拟通道（VC 0-7）**。 |
| **HdrFC** | Byte 1, [5:0]<br>Byte 2, [7:6] | 包含指定虚拟通道的头部存储信用计数。每个信用代表1个头部+可选TLP摘要（ECRC）的空间。 |
| **DataFC** | Byte 2, [3:0]<br>Byte 3, [7:0] | 包含指定虚拟通道的数据存储信用计数。每个信用代表**16字节**。 |
| **16位CRC** | Byte 4, [7:0]<br>Byte 5, [7:0] | 保护此DLLP内容的16位CRC。计算基于字节0-3，无论是否使用所有字段。 |

*[原文:第314-316页]*

---

## 9.9 厂商特定DLLP格式（Vendor-Specific DLLP Format）

最后定义的DLLP类型用于**厂商特定用途**。因此，只有**DLLP类型字段**由规范定义（**0011 0000b**），剩余内容可供厂商定义使用。

*[原文:第316页]*

---

## 9.10 Linux内核中的DLLP支持

在Linux内核中，DLLP（数据链路层包）的处理主要由PCIe控制器硬件完成，内核通过配置寄存器和错误报告机制与数据链路层交互。

### 9.10.1 DLLP错误检测

**文件**：`drivers/pci/pcie/aer.c`

内核通过AER（Advanced Error Reporting）机制检测DLLP相关错误：

```c
// 可纠正错误：Bad DLLP
#define PCI_ERR_COR_BAD_DLLP    0x00000080

// AER错误字符串
static const char *aer_correctable_error_string[] = {
    ...
    "BadDLLP",            /* Bit Position 7 */
    ...
};
```

当硬件检测到CRC错误的DLLP时，会设置此标志并触发AER中断，内核记录该错误。

### 9.10.2 流量控制DLLP与内核

流量控制DLLP（InitFC1/InitFC2/UpdateFC）完全由硬件自动处理：

1. **初始化阶段**：硬件自动交换InitFC1和InitFC2 DLLP完成流量控制初始化
2. **运行时更新**：硬件自动发送UpdateFC DLLP更新信用值
3. **内核可见性**：内核不直接参与流量控制DLLP的交换

### 9.10.3 电源管理DLLP

**文件**：`drivers/pci/pcie/aspm.c`

电源管理DLLP（PM_Enter_L1/L23等）与ASPM（Active State Power Management）相关：

```c
// ASPM L1状态转换涉及PM DLLP交换
static int pcie_set_aspm(struct pci_dev *pdev, u16 value)
{
    // 配置ASPM会触发硬件发送PM DLLP
}
```

### 9.10.4 厂商特定DLLP

厂商特定DLLP（Vendor Specific DLLP）由设备厂商定义，内核通常不处理这类DLLP，除非设备驱动显式支持。

### 9.10.5 调试接口

内核提供以下方式查看DLLP相关状态：

```bash
# 查看AER错误统计（包含Bad DLLP计数）
$ cat /sys/bus/pci/devices/<device>/aer_dev_stats

# 查看PCIe链路能力（包含流量控制相关）
$ lspci -vv -s <device> | grep -A 20 "LnkCap"

# 查看设备状态
$ cat /sys/kernel/debug/pcie/devices/<device>/status
```

### 9.10.6 内核与DLLP的交互总结

| DLLP类型 | 内核参与程度 | 说明 |
|----------|--------------|------|
| Ack/Nak | 无 | 硬件自动处理 |
| 流量控制 | 无 | 硬件自动处理 |
| 电源管理 | 间接 | 通过ASPM配置触发 |
| 厂商特定 | 可选 | 设备驱动特定 |
| 错误DLLP | 监控 | 通过AER报告 |

[内核源码参考: Linux 5.15+]

---

## 参考文档引用详情表

| 引用标记 | 文档来源 | 章节/页码 |
|----------|----------|----------|
| [原文:第307-308页] | 《PCI Express Technology 3.0》 | 第9章，第307-308页 |
| [原文:第308页] | 《PCI Express Technology 3.0》 | 第9章，第308页 |
| [原文:第309页] | 《PCI Express Technology 3.0》 | 第9章，第309页 |
| [原文:第309-310页] | 《PCI Express Technology 3.0》 | 第9章，第309-310页 |
| [原文:第310-311页] | 《PCI Express Technology 3.0》 | 第9章，第310-311页 |
| [原文:第311-312页] | 《PCI Express Technology 3.0》 | 第9章，第311-312页 |
| [原文:第312-313页] | 《PCI Express Technology 3.0》 | 第9章，第312-313页 |
| [原文:第313-314页] | 《PCI Express Technology 3.0》 | 第9章，第313-314页 |
| [原文:第314-316页] | 《PCI Express Technology 3.0》 | 第9章，第314-316页 |
| [原文:第316页] | 《PCI Express Technology 3.0》 | 第9章，第316页 |

---

## 术语对照表

| 英文术语 | 中文翻译 |
|----------|----------|
| DLLP (Data Link Layer Packet) | 数据链路层包 |
| TLP (Transaction Layer Packet) | 事务层包 |
| Ack (Acknowledge) | 确认 |
| Nak (Negative Acknowledge) | 否定确认 |
| VC (Virtual Channel) | 虚拟通道 |
| Flow Control | 流量控制 |
| Credit | 信用 |
| Posted Request | Posted请求 |
| Non-Posted Request | Non-Posted请求 |
| Completion | 完成 |
| CRC (Cyclic Redundancy Check) | 循环冗余校验 |
| LCRC (Link CRC) | 链路CRC |
| SDP (Start of DLLP) | DLLP起始 |
| END (End Good) | 良好结束 |
| Power Management | 电源管理 |
| Physical Layer | 物理层 |
| Data Link Layer | 数据链路层 |
| Transaction Layer | 事务层 |
| 8b/10b Encoding | 8b/10b编码 |
| 128b/130b Encoding | 128b/130b编码 |
| Header FC | 头部流量控制 |
| Data FC | 数据流量控制 |
| InitFC | 初始化流量控制 |
| UpdateFC | 更新流量控制 |
| Vendor Specific | 厂商特定 |
| Sequence Number | 序列号 |
| NEXT_RCV_SEQ | 下一接收序列号 |
| Symbol Time | 符号时间 |
| Bad DLLP | 错误DLLP |
| AER (Advanced Error Reporting) | 高级错误报告 |
| ASPM (Active State Power Management) | 主动状态电源管理 |

---

*翻译完成于2026年3月17日*
*校验更新于2026年3月17日*

---

## 本章图片附录

以下是本章相关的原文图片：

### 图9-1.DLLP格式.jpg

![图9-1.DLLP格式.jpg](../images/图9-1.DLLP格式.jpg)

### 图9-2.Ack_DLLP.jpg

![图9-2.Ack_DLLP.jpg](../images/图9-2.Ack_DLLP.jpg)

### 图9-3.Nak_DLLP.jpg

![图9-3.Nak_DLLP.jpg](../images/图9-3.Nak_DLLP.jpg)

