# 第16章 电源管理 (Power Management)

> **来源**: MindShare《PCI Express Technology 3.0》第16章  
> **原文页码**: 第703-793页  
> **翻译日期**: 2026-03-17

---

## 章节导航

- **上一章**: 第15章 错误检测与处理 (Error Detection and Handling)
- **下一章**: 第17章 中断 (Interrupts)

---

## 上一章回顾

上一章讨论了PCIe端口或链路中发生的错误类型、如何检测和报告这些错误，以及处理这些错误的选项。由于PCIe被设计为与PCI错误报告向后兼容，本章首先回顾了PCI处理错误的方法作为背景信息，然后重点介绍PCIe对可纠正错误、非致命错误和致命错误的处理。

## 本章内容

本章为系统电源管理讨论提供整体背景，并详细描述与PCI总线电源管理接口规范（PCI Bus PM Interface Spec）和高级配置与电源接口（ACPI）兼容的PCIe电源管理。PCIe定义了PCI-PM规范的扩展，主要关注链路电源和事件管理。本章还概述了OnNow计划、ACPI以及Windows操作系统的参与。

## 下一章预告

下一章详细介绍PCIe功能生成中断的不同方式。旧的PCI模型使用引脚来实现中断，但在串行模型中，边带信号是不可取的，因此支持带内MSI（消息信号中断）机制被设为强制要求。为了支持传统系统，PCI INTx#引脚操作仍可通过PCIe INTx消息进行仿真。本章将描述PCI传统INTx#方法和较新的MSI/MSI-X版本。

---

## 16.1 引言 (Introduction)

PCI Express电源管理（PM）定义了四个主要支持领域：

### 16.1.1 PCI兼容电源管理 (PCI-Compatible PM)

PCIe电源管理在硬件和软件上与PCI-PM和ACPI规范兼容。这项支持要求所有功能都包含PCI电源管理能力寄存器，允许软件通过配置请求在软件控制下将功能在PM状态之间转换。在2.1规范版本中，通过添加动态电源分配（DPA）进行了修改，这是另一组寄存器，为D0电源状态添加了若干子状态，为软件提供更细粒度的PM机制。

### 16.1.2 原生PCIe扩展 (Native PCIe Extensions)

这些扩展定义了用于链路的自主、基于硬件的主动状态电源管理（ASPM），以及唤醒系统的机制、报告电源管理事件（PME）的消息事务，以及计算和报告从低功耗状态到活动状态延迟的方法。

### 16.1.3 带宽管理 (Bandwidth Management)

2.1规范版本添加了硬件自动更改链路宽度或链路数据速率或两者的能力，以改善功耗。这允许在需要时提供高性能，在可接受较低性能时保持低功耗。尽管带宽管理被视为电源管理主题，但我们在"链路初始化与训练"一章的第618页"动态带宽更改"部分描述此功能，因为它涉及LTSSM（链路训练与状态状态机）。

### 16.1.4 事件时序优化 (Event Timing Optimization)

不考虑系统电源状态而启动总线主控事件或中断的外设会导致其他系统组件保持在高功耗状态以服务它们，从而导致比必要情况更高的功耗。2.1规范通过添加两种新机制纠正了这一缺陷：

- **优化缓冲区刷新和填充（OBFF）**：让系统通知外设当前系统电源状态
- **延迟容限报告（LTR）**：允许设备报告它们当前可以容忍的服务延迟


---

## 本章结构

本章分为以下几个主要部分：

1. **第一部分**是电源管理入门，涵盖系统软件在控制电源管理功能方面的作用。此讨论仅考虑Windows操作系统的视角，因为它是PC最常见的操作系统，其他操作系统不作描述。

2. **第二部分"功能电源管理"**（第713页）讨论使用PCI-PM能力寄存器将功能置于低功耗设备状态的方法。请注意，某些寄存器定义被PCIe功能修改或未使用。

3. **"主动状态电源管理（ASPM）"**（第735页）描述基于硬件的自主链路电源管理。软件确定为环境启用哪个级别的ASPM，可能通过读取该功能将产生的恢复延迟值，但之后电源转换的时序由硬件控制。软件不控制转换，也无法查看链路处于哪个电源状态。

4. **"软件启动的链路电源管理"**（第760页）讨论软件更改设备电源状态时强制执行的链路电源管理。

5. **"链路唤醒协议和PME生成"**（第768页）描述设备如何请求软件将它们返回到活动状态以便服务事件。当电源已从设备移除时，如果设备要监控事件并向系统发出唤醒信号以恢复电源并重新激活链路，则必须有辅助电源可用。

6. 最后，描述事件时序功能，包括OBFF和LTR。

---

## 16.2 电源管理入门 (Power Management Primer)

PCI总线PM接口规范描述了PCIe所需的电源管理寄存器。这些允许操作系统直接管理功能的电源环境。在深入详细描述之前，让我们先描述此功能在系统整体背景中的位置。

### 16.2.1 PCI PM基础 (Basics of PCI PM)

本节概述Windows操作系统如何与其他主要软件和硬件元素交互，以管理各个设备和整个系统的功耗。表16-1介绍了此过程中涉及的主要元素，并提供了它们如何相互关联的基本描述。应注意，PCI电源管理规范和ACPI规范都没有规定操作系统使用的PM策略。但是，它们确实定义了用于控制功能功耗的寄存器（和一些数据结构）。

**表16-1：PC电源管理中涉及的主要软件/硬件元素**

| 元素 | 职责 |
|------|------|
| **OS（操作系统）** | 通过向ACPI驱动程序、设备驱动程序和PCI Express总线驱动程序发送请求来指导整体系统电源管理。具有电源节省意识的应用程序与操作系统交互以完成设备电源管理。 |
| **ACPI驱动程序** | 管理不符合行业标准的嵌入式系统设备的配置、电源管理和热控制。例如包括芯片组特定寄存器、控制电源平面的系统板特定寄存器等。PCIe功能内的PM寄存器（嵌入式或其他）由PCI PM规范定义，因此不由ACPI驱动程序管理，而是由PCI Express总线驱动程序管理。 |
| **设备驱动程序** | 类驱动程序可以与属于其被编写控制的设备类中的任何设备一起工作。设备驱动程序也不理解特定设备类型的特定总线实现所特有的设备特性。例如，它不会理解PCIe功能的配置寄存器集。PCI Express总线驱动程序是与这些寄存器通信的实体。当它从操作系统接收到控制PCIe设备电源状态的请求时，它会将请求传递给PCI Express总线驱动程序。 |
| **小型端口驱动程序** | 由设备供应商提供，它从类驱动程序接收请求，并将它们转换为对设备寄存器集进行适当的一系列访问。 |
| **PCI Express总线驱动程序** | 此驱动程序对所有PCI Express兼容设备都是通用的。它管理它们的电源状态和配置寄存器。它从设备驱动程序接收请求以更改设备电源管理逻辑的状态。 |
| **每个功能配置空间内的PCI Express PM寄存器** | 这些寄存器的位置、格式和用法由PCIe规范定义。PCI Express总线驱动程序理解此规范，因此是在功能的设备驱动程序请求时负责访问功能PM寄存器的实体。 |
| **系统板电源平面和总线时钟控制逻辑** | 此逻辑的实现和控制通常是系统板设计特定的，因此由ACPI驱动程序控制（在操作系统指导下）。 |

### 16.2.2 ACPI规范定义整体PM (ACPI Spec Defines Overall PM)

ACPI（高级配置与电源接口）规范是几年前由多家公司联合编写的，旨在为计算平台提供OSPM（操作系统级电源管理）的行业标准。ACPI通过定义系统电源状态、硬件寄存器和软件交互来完成基于操作系统的电源管理。

#### 16.2.2.1 系统PM状态 (System PM States)

**表16-2：OnNow设计计划定义的系统PM状态**

| 电源状态 | 描述 |
|----------|------|
| **工作 (G0/S0)** | 系统完全运行。 |
| **睡眠 (G1)** | 系统看起来已关闭，功耗已降低。返回"工作"状态所需的时间与所选电源节省级别成反比。<br>• **S1** - 缓存刷新，CPU停止<br>• **S2** - 与S1相同，只是现在CPU已断电。不常用。<br>• **S3** - （也称为"挂起到RAM"或"待机"）系统上下文保存在内存中，更多系统被关闭。<br>• **S4** - （也称为"挂起到磁盘"或"休眠"）系统上下文复制到磁盘，然后从系统移除电源。 |
| **软关闭 (G2/S5)** | 系统看起来已关闭，功耗最小。需要完全重新引导才能返回"工作"状态。 |
| **机械关闭 (G3)** | 系统已与所有电源断开连接，没有电源可用。 |

#### 16.2.2.2 设备PM状态 (Device PM States)

**表16-3：OnNow定义的设备级PM状态**

| 状态 | 描述 |
|------|------|
| **D0** | 强制要求。设备完全运行，使用系统的全功率。2.1规范版本添加了DPA（动态电源分配）寄存器以支持32个子状态。 |
| **D1** | 可选。低功耗状态，设备上下文可能丢失也可能不丢失。 |
| **D2** | 可选。比D1更低的功耗状态，可实现更大的电源节省，但会产生更长的恢复延迟。 |
| **D3** | 强制要求。设备准备断电，上下文可能丢失。恢复时间将比D2长。 |

#### 16.2.2.3 设备上下文定义 (Definition of Device Context)

**一般上下文**包括：
- 其配置寄存器的内容
- 其本地内存和IO寄存器的状态
- 如果它包含处理器，则其当前程序指针和其他寄存器的内容

**PME上下文**包括：
- PME消息能力
- PME启用/禁用控制位
- PME状态位
- 设备特定控制位和状态位


---

## 16.3 功能电源管理 (Function Power Management)

PCI Express功能需要支持电源管理，并且必须实现几个寄存器和相关位字段，如下所述。

### 16.3.1 PM能力寄存器集 (The PM Capability Register Set)

PCI-PM规范定义了电源管理能力配置寄存器。这些寄存器在PCI中是可选的，但在PCIe中是必需的，位于PCI兼容配置空间中，能力ID为01h。

**图16-2：PCI电源管理能力寄存器集**

![ASPM](../images/图16-2.ASPM.png)

*原文图示：ASPM*


```
  31        16 15       8 7        0
 +------------+----------+----------+
 | Power Mgmt | Pointer to| Capability|
 | Capabilities| Next Cap | ID = 01h |
 | (PMC)      |          |          |
 +------------+----------+----------+
 | Data Reg   | Bridge Support | PM Control/Status |
 |            | Extensions     | (PMCSR)           |
 +------------+----------------+-------------------+
```

### 16.3.2 设备PM状态 (Device PM States)

每个PCI Express功能必须支持全开D0状态和全关D3状态，而D1和D2是可选的。

#### 16.3.2.1 D0状态—全开 (D0 State—Full On)

**强制要求**。在此状态下，不生效电源节省，设备完全运行。所有PCIe功能必须支持D0状态，技术上有两个子状态：D0未初始化和D0活动。

**D0未初始化**：功能在基本复位后进入D0未初始化，或在某些情况下，当软件将其从D3hot转换到D0时。通常，寄存器返回到其默认状态。

**D0活动**：一旦功能被软件配置和启用，它就处于D0活动状态并完全运行。

**表16-5：D0电源管理策略**

| 链路PM状态 | 功能PM状态 | 必须有效的寄存器或状态 | 电源 | 允许对功能执行的操作 | 功能允许的操作 |
|------------|-----------|----------------------|------|---------------------|---------------|
| L0 | D0未初始化 | PME上下文 | < 10W | PCIe配置事务 | 无 |
| L0/L0s/L1 | D0活动 | 全部 | 满 | 任何PCIe事务 | 任何事务、中断或PME |

#### 16.3.2.2 动态电源分配(DPA) (Dynamic Power Allocation)

**可选**。2.1版本的基本规范添加了另一个可选能力，为D0定义了32个更多子状态并描述其特性。这旨在促进设备驱动程序、操作系统和执行应用程序之间关于电源管理的协商。

DPA寄存器仅在设备电源状态处于D0时适用，在D1-D3状态中不适用。最多可定义32个子状态，必须从0到最大值连续编号。

#### 16.3.2.3 D1状态—浅睡眠 (D1 State—Light Sleep)

**可选**。在进入此状态之前，软件必须确保所有未完成的非发布请求已收到其关联的完成。在此轻电源节省状态下，功能不会启动请求（如果启用，PME消息除外）。

**D1状态特性**：
- 当设备进入D1状态时，链路被强制到L1电源状态
- 在此状态下接受配置和消息请求，但所有其他请求必须作为不支持的请求处理
- 功能可以重新激活链路并发送PME消息（如果在此状态下支持并启用）
- 功能可能会或可能不会丢失其上下文

#### 16.3.2.4 D2状态—深睡眠 (D2 State—Deep Sleep)

**可选**。此电源状态提供比D1更深的电源节省，但比D3hot状态少。与D1一样，功能不会启动请求（PME消息除外）或作为除配置外请求的目标。

**D2状态特性**：
- 当设备转换到D2状态时，链路状态必须转换到L1
- 在此状态下接受配置和消息请求
- 功能可以发送PME消息（如果支持并启用）
- 功能可能会或可能不会丢失其上下文

#### 16.3.2.5 D3状态—全关 (D3 State—Full Off)

**强制要求**。所有功能必须支持D3状态。这是最深的状态，电源节省最大化。当软件将此电源状态写入设备时，它进入D3hot状态，意味着电源仍然施加。从设备移除电源（Vcc）将其置于D3cold状态。

**D3hot状态**：
- 软件通过写入其电源管理控制状态寄存器（PMCSR）的PowerState字段将功能置于D3hot
- 在此状态下，功能只能启动PME或PME_TO_ACK消息，并且只能响应配置请求或PME_Turn_Off消息
- 当功能更改为D3hot时，链路被强制到L1状态
- 功能上下文可能会丢失

**D3cold状态**：
- 每个PCI Express功能在从功能移除电源（Vcc）时进入D3cold PM状态
- 当电源恢复时，设备必须复位或生成内部复位，将其从D3cold带到D0未初始化
- 能够生成PME的功能必须在此状态下保持PME上下文

### 16.3.3 功能PM状态转换 (Function PM State Transitions)

**图16-6：PCIe功能D状态转换**

```
                    Power On/Reset
                          |
                          v
                   +-------------+
                   | D0 Uninit   |
                   +-------------+
                          |
                    (Configured)
                          |
                          v
                   +-------------+        +-------------+
                   |   D0 Active |<------>|     D1      |
                   +-------------+        +-------------+
                          |                     |
                          |              +-------------+
                          |------------->|     D2      |
                          |              +-------------+
                          |                     |
                          |              +-------------+
                          |------------->|   D3hot     |
                          |              +-------------+
                          |                     |
                          |              +-------------+
                          |------------->|   D3cold    |
                          |              | (Vcc Removed|
                          |              +-------------+
                          v
```

**表16-10：功能状态转换描述**

| 从状态 | 到状态 | 描述 |
|--------|--------|------|
| D0未初始化 | D0活动 | 功能已被其驱动程序完全配置和启用 |
| D0未初始化 | D1 | 软件将PMCSR PowerState写入D1 |
| D0未初始化 | D2 | 软件将PMCSR PowerState写入D2 |
| D0未初始化 | D3hot | 软件将PMCSR PowerState写入D3hot |
| D0活动 | D0未初始化 | 软件将PMCSR PowerState写入D0 |
| D0活动 | D2 | 软件将PMCSR PowerState写入D2 |
| D0活动 | D3hot | 软件将PMCSR PowerState写入D3hot |
| D1 | D0活动 | 软件将PMCSR PowerState写入D0 |
| D1 | D3hot | 软件将PMCSR PowerState写入D3hot |
| D2 | D0活动 | 软件将PMCSR PowerState写入D0 |
| D2 | D3hot | 软件将PMCSR PowerState写入D3hot |
| D3hot | D3cold | 电源从功能移除 |
| D3hot | D0未初始化 | 软件将PMCSR PowerState写入D0 |
| D3cold | D0未初始化 | 电源恢复到功能 |


---

## 16.4 PCI-PM寄存器详细描述 (Detailed Description of PCI-PM Registers)

PCI总线PM接口规范定义了在PCIe功能中实现的PM寄存器。配置软件可以确定PM能力并控制其属性。

### 16.4.1 PM能力(PMC)寄存器 (PM Capabilities Register)

此16位只读寄存器的字段如表16-12所述。

**表16-12：PMC寄存器位分配**

| 位 | 描述 |
|----|------|
| **31:27** | **PME_Support字段**。指示功能在哪些PM状态能够发送PME消息。位中的零表示在相应的PM状态不支持PME通知。<br>位27=D0, 28=D1, 29=D2, 30=D3hot, 31=D3cold |
| **26** | **D2_Support位**。1=功能支持D2 PM状态 |
| **25** | **D1_Support位**。1=功能支持D1 PM状态 |
| **24:22** | **Aux_Current字段**。对于支持从D3cold状态生成PME消息的功能，此字段报告功能保留PME上下文信息的逻辑对3.3Vaux电源的电流需求 |
| **21** | **DSI位**。1表示进入D0未初始化状态后，功能需要除PCI配置头寄存器设置之外的额外配置 |
| **20** | 保留 |
| **19** | **PME Clock位**。不适用于PCI Express，必须硬连线为0 |
| **18:16** | **Version字段**。指示功能符合的PCI总线PM接口规范版本。001=1.0, 010=1.1（PCI Express要求） |

### 16.4.2 PM控制/状态寄存器(PMCSR) (PM Control and Status Register)

此寄存器对所有PCI Express设备都是必需的，具有以下用途：
- 如果功能实现PME能力，PME Enable位允许软件启用或禁用功能断言PME消息或WAKE#信号的能力
- PME Status位反映PME是否已发生
- PowerState字段可用于读取功能的当前PM状态或写入以将功能置于新PM状态

**表16-13：PM控制/状态寄存器(PMCSR)位分配**

| 位 | 复位值 | 读/写 | 描述 |
|----|--------|-------|------|
| **31:24** | 全零 | 只读 | 见"数据寄存器" |
| **23** | 零 | 只读 | PCI Express中未使用 |
| **22** | 零 | 只读 | PCI Express中未使用 |
| **21:16** | 全零 | 只读 | 保留 |
| **15** | 见描述 | RW1CS | **PME_Status位**。反映功能是否经历了PME。软件通过写入1来清除此位。 |
| **14:13** | 设备特定 | 只读 | **Data_Scale字段**。数据寄存器的乘数 |
| **12:9** | 0000b | 读/写 | **Data_Select字段**。选择要通过数据寄存器查看的数据 |
| **8** | 见描述 | 读/写 | **PME_En位**。1=启用功能发送PME消息的能力；0=禁用 |
| **7:2** | 全零 | 只读 | 保留 |
| **1:0** | 00b | 读/写 | **PowerState字段**。软件使用此字段读取功能的当前PM状态或写入新PM状态。00=D0, 01=D1, 10=D2, 11=D3hot |

### 16.4.3 数据寄存器 (Data Register)

**可选，只读**。数据寄存器是一个8位只读寄存器，为软件提供以下信息：
- 所选PM状态下的功耗；用于电源预算
- 所选PM状态下的功率耗散；用于管理热环境

如果实现了数据寄存器，则还必须实现PMCSR寄存器的Data_Select和Data_Scale字段。

**表16-14：数据寄存器解释**

| Data Select值 | 数据寄存器中报告的数据 | Data_Scale字段解释 |
|---------------|----------------------|-------------------|
| 00h | D0中的功耗 | 00b=未知, 01b=×0.1, 10b=×0.01, 11b=×0.001 |
| 01h | D1中的功耗 | 单位：瓦特 |
| 02h | D2中的功耗 | |
| 03h | D3中的功耗 | |
| 04h | D0中的功率耗散 | |
| 05h | D1中的功率耗散 | |
| 06h | D2中的功率耗散 | |
| 07h | D3中的功率耗散 | |
| 08h | 多功能PCI设备中，Function 0指示封装中所有功能共用的逻辑功耗 | |

---

## 16.5 链路电源管理介绍 (Introduction to Link Power Management)

我们已经看到软件如何将设备置于几种设备电源状态之一，现在让我们考虑PCIe如何管理链路电源。设备电源和链路电源彼此相关，如表16-15所示。

**表16-15：设备与链路电源状态之间的关系**

| 下游组件D状态 | 允许的上游组件D状态 | 允许的互连状态 |
|--------------|-------------------|--------------|
| D0 | D0 | L0, L0s & L1（可选） |
| D1 | D0-D1 | L1 |
| D2 | D0-D2 | L1 |
| D3hot | D0-D3hot | L1, L2/L3 Ready |
| D3cold | D0-D3cold | L2（AUX电源）, L3 |

**D0** — 设备完全供电，通常在L0链路状态。通过使用DPA子状态和基于硬件的链路电源管理，可以在不离开此状态的情况下获得一些电源节省。

**D1 & D2** — 当软件将设备状态更改为D1或D2时，链路必须自动转换到L1状态。

**D3hot** — 当软件将设备置于D3状态时，链路自动转换到L1。软件现在可以选择移除参考时钟和电源，将设备置于D3cold。

**D3cold** — 在此状态下，主电源和参考时钟已关闭。但是，辅助电源（VAUX）可能可用，允许设备向系统发出唤醒事件信号。


---

## 16.6 主动状态电源管理(ASPM) (Active State Power Management)

主动状态电源管理（ASPM）是PCIe中基于硬件的自主链路电源管理机制。与软件控制的电源管理不同，ASPM允许链路在没有软件干预的情况下自动进入和退出低功耗状态。

### 16.6.1 ASPM概述

ASPM允许链路在空闲时自动进入低功耗状态以节省电源，并在需要传输数据时快速返回活动状态。软件确定启用哪个级别的ASPM，但之后的电源转换时序由硬件控制。

**ASPM的两个主要级别**：
- **L0s**：一种较浅的低功耗状态，具有更快的退出延迟
- **L1**：一种更深的低功耗状态，具有更长的退出延迟但节省更多电源

### 16.6.2 ASPM L0s状态

L0s是一种低功耗状态，其中发送器进入电气空闲以节省电源，而接收器保持活动状态以检测传入流量。

**L0s特性**：
- 进入L0s不需要握手
- 当发送器没有要发送的数据时，它可以单方面进入L0s
- 退出L0s由接收器检测到的传入流量触发
- 退出延迟很短（通常<1μs）

### 16.6.3 ASPM L1状态

L1是一种更深的低功耗状态，其中发送器和接收器都进入电气空闲。

**L1特性**：
- 进入L1需要链路伙伴之间的握手
- 当两个链路伙伴都同意进入L1时，链路转换到L1
- 退出L1需要更长的恢复时间
- 比L0s节省更多电源

### 16.6.4 ASPM配置

ASPM通过PCI Express能力寄存器中的Link Control寄存器配置。

**Link Control寄存器中的ASPM控制位**：

| 位 | 字段 | 描述 |
|----|------|------|
| 1:0 | **Active State PM Control** | 控制ASPM级别：<br>00b = 禁用ASPM<br>01b = 启用L0s<br>10b = 启用L1<br>11b = 启用L0s和L1 |

### 16.6.5 ASPM与设备电源状态的关系

当设备处于D0状态时，ASPM可以独立于设备电源状态管理链路电源。然而，当设备转换到D1、D2或D3hot时，链路必须转换到L1（如果ASPM已启用）。

---

## 16.7 软件启动的链路电源管理 (Software Initiated Link Power Management)

除了ASPM之外，PCIe还支持软件启动的链路电源管理，当软件更改设备电源状态时强制执行。

### 16.7.1 软件控制的链路状态转换

当软件将设备置于低功耗状态时，链路必须相应地转换：

- **D0 → D1/D2/D3hot**：链路必须转换到L1
- **D3hot → D3cold**：链路准备进入L2/L3 Ready状态

### 16.7.2 L2/L3 Ready握手序列

在可以移除电源之前，系统必须启动握手过程以准备链路进入L2/L3 Ready状态。

**L2/L3 Ready握手序列**：
1. 系统广播PME_Turn_Off消息到所有设备
2. 设备以PME_TO_ACK消息响应
3. 设备发送PM_Enter_L23 DLLP
4. 链路进入L2/L3 Ready状态
5. 现在可以安全地移除电源和参考时钟

---

## 16.8 链路唤醒协议和PME生成 (Link Wake Protocol and PME Generation)

设备可能需要请求软件将它们返回到活动状态以服务事件。这就是电源管理事件（PME）机制。

### 16.8.1 PME概述

PME（电源管理事件）允许设备在需要服务时唤醒系统。这在设备处于低功耗状态但需要处理事件（如网络数据包到达或调制解调器来电）时特别有用。

### 16.8.2 PME消息

PME通过PCIe消息事务报告：

**PME消息格式**：
- **Msg类型**：PME消息是一种消息请求（Msg）
- **路由**：PME消息路由到根复合体
- **Requester ID**：发送PME消息的功能的BDF（总线、设备、功能）

### 16.8.3 PME从D3cold唤醒

从D3cold唤醒需要辅助电源（Vaux）来维护PME上下文和信号逻辑。

**D3cold唤醒方法**：
- **Beacon信号**：设备在链路上发送低功耗信标信号
- **WAKE#信号**：某些 form factor支持专用的WAKE#引脚

### 16.8.4 辅助电源 (Auxiliary Power)

辅助电源（3.3Vaux）是在主电源（Vcc）关闭时保持供电的电源。它用于：
- 维护PME上下文
- 监控唤醒事件
- 发送PME消息或信标信号

**Aux_Current字段**：PMC寄存器中的此字段报告功能从3.3Vaux电源汲取的电流。

---

## 16.9 事件时序优化 (Event Timing Optimization)

### 16.9.1 优化缓冲区刷新和填充(OBFF) (Optimized Buffer Flush and Fill)

OBFF允许系统通知外设当前系统电源状态，使外设能够相应地调整其行为。

**OBFF状态**：
- **OBFF Disabled**：设备正常操作
- **OBFF Enabled with System in CPU Active State**：系统完全运行，设备可以正常操作
- **OBFF Enabled with System in Memory Access State**：系统处于内存访问状态
- **OBFF Enabled with System in Idle State**：系统空闲，设备应避免生成流量

### 16.9.2 延迟容限报告(LTR) (Latency Tolerance Reporting)

LTR允许设备报告它们当前可以容忍的服务延迟。这使系统能够优化电源管理决策。

**LTR消息内容**：
- **Latency Value**：设备可以容忍的最大延迟
- **Latency Scale**：延迟值的缩放因子
- **Requirement位**：指示延迟要求是最大要求还是非最大要求

**LTR机制**：
- 设备通过LTR消息报告其延迟容限
- 交换机聚合下游设备的延迟容限
- 根端口使用此信息优化系统电源管理

---

## 16.10 Linux内核ASPM实现

### 源码位置
`drivers/pci/pcie/aspm.c`

### ASPM状态定义
```c
#define ASPM_STATE_L0S_UP  (1)   /* 上游方向L0s状态 */
#define ASPM_STATE_L0S_DW  (2)   /* 下游方向L0s状态 */
#define ASPM_STATE_L1      (4)   /* L1状态 */
#define ASPM_STATE_L1_1    (8)   /* ASPM L1.1状态 */
#define ASPM_STATE_L1_2    (0x10)/* ASPM L1.2状态 */
```

### 策略配置
内核支持四种ASPM策略：
- `performance` - 禁用ASPM，追求性能
- `powersave` - 启用L0s/L1，平衡功耗
- `powersupersave` - 启用所有节电功能
- `default` - 使用BIOS默认设置

### 关键函数

1. **延迟计算**（与PCIe规范一致）
   ```c
   static u32 calc_l0s_latency(u32 lnkcap);
   static u32 calc_l1_latency(u32 lnkcap);
   ```

2. **公共时钟配置**
   ```c
   static void pcie_aspm_configure_common_clock(struct pcie_link_state *link);
   ```

3. **链路重训练**
   ```c
   static int pcie_retrain_link(struct pcie_link_state *link);
   ```

### 用户接口

通过sysfs调整ASPM策略：
```bash
# 查看当前ASPM状态
cat /sys/bus/pci/devices/0000:00:01.0/power/aspm

# 设置ASPM策略（需root权限）
echo powersave > /sys/module/pcie_aspm/parameters/policy
```

### 时钟电源管理

```c
// 启用/禁用时钟请求
static void pcie_set_clkpm(struct pcie_link_state *link, int enable);
```
通过`PCI_EXP_LNKCTL_CLKREQ_EN`位控制。

---

## 16.11 总结 (Summary)

PCI Express电源管理提供了一套全面的机制来管理系统和设备的功耗：

1. **PCI兼容PM**：与PCI-PM和ACPI规范兼容，允许软件控制设备电源状态（D0-D3）

2. **ASPM**：基于硬件的自主链路电源管理，允许链路在空闲时自动进入低功耗状态（L0s/L1）

3. **PME机制**：允许设备请求系统返回活动状态以服务事件

4. **事件时序优化**：OBFF和LTR机制允许更好的系统电源管理协调

这些机制共同使PCIe系统能够在提供高性能的同时实现高效的电源管理。

---

## 术语对照表

| 英文术语 | 中文翻译 | 缩写 |
|----------|----------|------|
| Power Management | 电源管理 | PM |
| Active State Power Management | 主动状态电源管理 | ASPM |
| Power Management Event | 电源管理事件 | PME |
| Device Power State | 设备电源状态 | D-state |
| Link Power State | 链路电源状态 | L-state |
| Dynamic Power Allocation | 动态电源分配 | DPA |
| Optimized Buffer Flush and Fill | 优化缓冲区刷新和填充 | OBFF |
| Latency Tolerance Reporting | 延迟容限报告 | LTR |
| Auxiliary Power | 辅助电源 | Vaux |
| Configuration Space | 配置空间 | |
| Root Complex | 根复合体 | RC |
| Advanced Configuration and Power Interface | 高级配置与电源接口 | ACPI |

---

*翻译完成于 2026-03-17*  
*原文来源: MindShare PCI Express Technology 3.0, Chapter 16*

---

## 16.11 Linux内核实现参考（平台特定补充）

### 16.11.1 Linux ASPM实现

Linux内核的ASPM（Active State Power Management）驱动实现了书中第16章介绍的主动状态电源管理。

#### ASPM策略配置

```c
// 来自 drivers/pci/pcie/aspm.c

// ASPM策略枚举（对应书中16.5节）
enum pcie_aspm_policy {
    POLICY_DEFAULT,     // 默认策略（BIOS设置）
    POLICY_PERFORMANCE, // 性能优先（禁用ASPM）
    POLICY_POWERSAVE,   // 省电优先（启用L0s和L1）
    POLICY_POWER_SUPERSAVE, // 超级省电
};

static enum pcie_aspm_policy aspm_policy = POLICY_DEFAULT;
static const char *policy_str[] = {
    [POLICY_DEFAULT] = "default",
    [POLICY_PERFORMANCE] = "performance",
    [POLICY_POWERSAVE] = "powersave",
    [POLICY_POWER_SUPERSAVE] = "powersupersave",
};
```

**与书中16.5节的对应**：

书中ASPM策略：
- 禁用ASPM - 性能优先
- 启用L0s - 轻度省电
- 启用L1 - 深度省电
- 同时启用L0s和L1 - 最大省电

Linux实现：
- `POLICY_PERFORMANCE` - 禁用ASPM
- `POLICY_POWERSAVE` - 启用L0s和L1
- `POLICY_POWER_SUPERSAVE` - 启用所有省电模式

#### ASPM链路状态结构

```c
// PCIe链路状态结构（对应书中16.5节）
struct pcie_link_state {
    struct pci_dev *pdev;           // PCI设备
    struct pcie_link_state *parent; // 父链路
    struct list_head siblings;      // 兄弟链路
    struct list_head children;      // 子链路
    
    // ASPM支持状态（对应书中16.5.1节）
    u32 aspm_support:2;             // 支持的ASPM级别
    u32 aspm_enabled:2;             // 启用的ASPM级别
    u32 aspm_default:2;             // 默认ASPM级别
    
    // L0s配置（对应书中16.5.2节）
    u32 latency_l0s:4;              // L0s退出延迟
    u32 latency_l1:4;               // L1退出延迟
    
    // 时钟配置
    u32 clkpm_enabled:1;            // 时钟电源管理
    u32 clkpm_default:1;            // 默认时钟电源管理
};

// ASPM级别定义
#define ASPM_STATE_L0S  (1 << 0)    // L0s状态
#define ASPM_STATE_L1   (1 << 1)    // L1状态
#define ASPM_STATE_L1SS (1 << 2)    // L1子状态（PCIe 4.0）
```

**与书中16.4节的对应**：

书中链路电源状态：
- L0 - 正常工作
- L0s - 轻度省电
- L1 - 深度省电
- L2 - 完全关闭

Linux实现：
- `ASPM_STATE_L0S` - 对应书中L0s
- `ASPM_STATE_L1` - 对应书中L1
- `ASPM_STATE_L1SS` - PCIe 4.0 L1子状态

#### ASPM配置代码

```c
// ASPM配置（对应书中16.5节）
static void pcie_config_aspm_link(struct pcie_link_state *link, u32 state)
{
    struct pci_dev *child = link->pdev;
    struct pci_dev *parent = link->parent->pdev;
    u32 val, enable_mask;
    
    // 读取当前ASPM配置
    pcie_capability_read_dword(child, PCI_EXP_LNKCTL, &val);
    enable_mask = PCI_EXP_LNKCTL_ASPM_L0S | PCI_EXP_LNKCTL_ASPM_L1;
    
    // 清除当前ASPM设置
    val &= ~enable_mask;
    
    // 配置新的ASPM状态
    if (state & ASPM_STATE_L1) {
        // 启用L1（对应书中16.5.3节）
        val |= PCI_EXP_LNKCTL_ASPM_L1;
    } else if (state & ASPM_STATE_L0S) {
        // 启用L0s（对应书中16.5.2节）
        val |= PCI_EXP_LNKCTL_ASPM_L0S;
    }
    
    // 写入配置
    pcie_capability_write_dword(child, PCI_EXP_LNKCTL, val);
    pcie_capability_write_dword(parent, PCI_EXP_LNKCTL, val);
    
    link->aspm_enabled = state;
}

// ASPM初始化（对应书中16.5.1节）
static int pcie_init_aspm_link(struct pcie_link_state *link)
{
    struct pci_dev *child = link->pdev;
    struct pci_dev *parent = link->parent->pdev;
    u32 lnkcap, lnkctl;
    
    // 读取链路能力（对应书中16.5.1节）
    pcie_capability_read_dword(child, PCI_EXP_LNKCAP, &lnkcap);
    pcie_capability_read_dword(parent, PCI_EXP_LNKCAP, &lnkcap);
    
    // 获取支持的ASPM级别
    link->aspm_support = (lnkcap & PCI_EXP_LNKCAP_ASPM) >> 10;
    
    // 获取退出延迟（对应书中16.5.2/16.5.3节）
    link->latency_l0s = (lnkcap & PCI_EXP_LNKCAP_L0SEL) >> 12;
    link->latency_l1 = (lnkcap & PCI_EXP_LNKCAP_L1EL) >> 15;
    
    // 配置ASPM
    pcie_config_aspm_link(link, policy_to_aspm_state(link));
    
    return 0;
}
```

**与书中16.5节的对应**：

书中ASPM配置流程：
1. 读取链路能力（L0s/L1支持、退出延迟）
2. 根据策略配置ASPM
3. 写入链路控制寄存器

Linux实现：
1. `pcie_capability_read_dword(PCI_EXP_LNKCAP)` - 读取能力
2. `policy_to_aspm_state()` - 策略转换
3. `pcie_capability_write_dword(PCI_EXP_LNKCTL)` - 写入控制

#### sysfs接口

```c
// ASPM sysfs接口（用户空间控制）
static ssize_t aspm_store(struct device *dev, struct device_attribute *attr,
                          const char *buf, size_t count)
{
    // 解析用户输入的策略
    if (!strncmp(buf, "performance", 11))
        aspm_policy = POLICY_PERFORMANCE;
    else if (!strncmp(buf, "powersave", 9))
        aspm_policy = POLICY_POWERSAVE;
    else if (!strncmp(buf, "default", 7))
        aspm_policy = POLICY_DEFAULT;
    
    // 重新配置所有链路
    pcie_update_aspm_policy();
    
    return count;
}

// 使用示例：
// echo "powersave" > /sys/module/pcie_aspm/parameters/policy
// echo "performance" > /sys/module/pcie_aspm/parameters/policy
```

### 16.11.2 设备电源状态管理

```c
// 设备电源状态设置（对应书中16.3节）
int pci_set_power_state(struct pci_dev *dev, pci_power_t state)
{
    u16 pmcsr, pmc;
    
    // 读取电源管理能力
    pci_read_config_word(dev, dev->pm_cap + PCI_PM_PMC, &pmc);
    
    // 检查状态支持
    if (state == PCI_D3hot && !(pmc & PCI_PM_CAP_D3))
        return -EIO;
    if (state == PCI_D3cold && !(pmc & PCI_PM_CAP_D3))
        return -EIO;
    
    // 读取当前状态
    pci_read_config_word(dev, dev->pm_cap + PCI_PM_CTRL, &pmcsr);
    
    // 设置新状态（对应书中16.3节表16-1）
    pmcsr &= ~PCI_PM_CTRL_STATE_MASK;
    pmcsr |= state;
    
    // 写入电源管理控制/状态寄存器
    pci_write_config_word(dev, dev->pm_cap + PCI_PM_CTRL, pmcsr);
    
    dev->current_state = state;
    
    return 0;
}

// 电源状态保存和恢复（对应书中16.3节）
int pci_save_state(struct pci_dev *dev)
{
    // 保存配置空间
    pci_read_config_dword(dev, PCI_VENDOR_ID, &dev->saved_config_space[0]);
    // ... 保存其他寄存器
    return 0;
}

int pci_restore_state(struct pci_dev *dev)
{
    // 恢复配置空间
    pci_write_config_dword(dev, PCI_VENDOR_ID, dev->saved_config_space[0]);
    // ... 恢复其他寄存器
    return 0;
}
```

**与书中16.3节的对应**：

书中设备电源状态：
- D0 - 正常工作
- D1 - 轻度省电
- D2 - 中度省电
- D3hot - 深度省电
- D3cold - 完全关闭

Linux实现：
- `PCI_D0` - D0状态
- `PCI_D1` - D1状态
- `PCI_D2` - D2状态
- `PCI_D3hot` - D3hot状态
- `PCI_D3cold` - D3cold状态

### 16.11.3 实际应用建议

**ASPM调试**：
1. 查看当前ASPM策略：`cat /sys/module/pcie_aspm/parameters/policy`
2. 查看链路ASPM状态：`lspci -vv | grep -i aspm`
3. 修改ASPM策略：`echo powersave > /sys/module/pcie_aspm/parameters/policy`

**电源状态调试**：
1. 查看设备电源状态：`cat /sys/bus/pci/devices/.../power_state`
2. 设置设备电源状态：`echo 3 > /sys/bus/pci/devices/.../power/control`

**性能优化**：
- 需要低延迟：使用`performance`策略（禁用ASPM）
- 需要省电：使用`powersave`策略（启用L0s和L1）
- 平衡模式：使用`default`策略

**故障排查**：
- ASPM导致设备不稳定：切换到`performance`策略
- 设备无法唤醒：检查唤醒能力配置
- 电源状态切换失败：检查设备是否支持该状态

---

*翻译完成于 2026-03-17*
*原文来源: MindShare PCI Express Technology 3.0, Chapter 16*
*平台补充: Linux ASPM实现参考*


---

## 本章图片附录

以下是本章相关的原文图片：


### 图16-1.电源状态.png

![图16-1.电源状态.png](../images/图16-1.电源状态.png)


### 图16-2.ASPM.png

![图16-2.ASPM.png](../images/图16-2.ASPM.png)


### 图16-3.L1子状态.png

![图16-3.L1子状态.png](../images/图16-3.L1子状态.png)

