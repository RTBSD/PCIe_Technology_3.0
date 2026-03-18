# 第17章 中断支持 (Interrupt Support)

> 来源: MindShare《PCI Express Technology 3.0》第17章
> 原文页码: 第793-831页

---

## 章节导航

**前一章**
前一章提供了系统电源管理的整体背景，并详细描述了PCIe电源管理，它与PCI总线电源管理接口规范(PCI Bus PM Interface Spec)和高级配置与电源接口(ACPI)规范兼容。PCIe定义了PCI-PM规范的扩展，主要关注链路电源和事件管理。同时还概述了OnNow计划、ACPI以及Windows操作系统的参与。

**本章**
本章描述了PCIe功能(Function)生成中断的不同方式。旧的PCI模型使用引脚来实现中断，但在串行模型中，边带信号是不受欢迎的，因此将对带内MSI(Message Signaled Interrupt，消息信号中断)机制的支持设为强制要求。为了软件向后兼容，PCI INTx#引脚操作仍可使用PCIe INTx消息进行模拟。本章将介绍PCI传统INTx#方法和较新版本的MSI/MSI-X。

**下一章**
下一章描述了PCIe定义的三种复位类型：基本复位(包括冷复位和热复位)、热复位(Hot Reset)和功能级复位(FLR)。讨论了使用边带复位信号PERST#生成系统复位的方法，以及基于带内TS1的热复位。

---

## 17.1 中断支持背景

### 17.1.1 概述

PCI架构支持来自外设的中断，作为提高其性能和减轻CPU轮询设备以确定何时需要服务的手段。PCIe基本不变地继承了PCI的这一支持，允许软件与PCI保持向后兼容。我们在本章提供系统中断处理的背景，但希望了解更多中断细节的读者可以参考以下资料：

- 关于PCI中断背景，请参阅PCI规范3.0版或MindShare教科书《PCI System Architecture》第14章(www.mindshare.com)
- 要了解本地APIC和IO APIC的更多信息，请参阅MindShare教科书《x86 Instruction Set Architecture》

### 17.1.2 两种中断传递方法

PCI使用路由到中央中断控制器的边带中断线。这种方法在简单的单CPU系统中工作良好，但存在一些缺点，促使人们转向一种称为MSI(Message Signaled Interrupts，消息信号中断)的新方法，并带有称为MSI-X(eXtended，扩展)的扩展。

**传统PCI中断传递** — 这种为PCI总线定义的原始机制每个设备最多包含四个信号或INTx#(INTA#、INTB#、INTC#和INTD#)，如图17-1所示。在这种模型中，引脚通过线或(wire-ORing)连接在一起，最终连接到8259 PIC(可编程中断控制器)的输入端。当引脚被置位时，PIC反过来向CPU置位其中断请求引脚，这是"传统模型"(第796页)中描述的过程的一部分。

![INTx](../images/图17-1.INTx.png)

*原文图示：INTx*


PCIe支持这种PCI中断功能以实现向后兼容，但串行传输的设计目标是最小化引脚数量。因此，INTx#信号没有实现为边带引脚。相反，功能(Function)可以生成带内中断消息包来指示引脚的置位或解除置位。这些消息充当"虚拟线"，目标指向系统中的中断控制器(通常在根复合体Root Complex中)，如图17-2所示(第796页)。该图还说明了使用引脚的旧PCI设备如何在PCIe系统中工作；桥将引脚的置位转换为向上游发送到根复合体的中断模拟消息(INTx)。

![MSI](../images/图17-2.MSI.png)

*原文图示：MSI*


预期PCIe设备通常不需要使用INTx消息，但在撰写本文时，实践中它们经常使用，因为系统软件尚未更新以支持MSI。

**MSI中断传递** — MSI通过使用存储器写操作来传递中断通知，从而消除了对边带信号的需求。"Message Signaled Interrupt"这个术语可能会令人困惑，因为它的名称中包含"Message"这个词，而Message是PCIe中的一种TLP类型，但MSI中断是一种Posted Memory Write(发布的存储器写)，而不是Message事务。MSI存储器写与其他存储器写的区别仅在于它们所针对的地址，这些地址通常由系统保留用于中断传递(例如，基于x86的系统传统上将地址范围FEEx_xxxxh保留用于中断传递)。

图17-2说明了来自各种类型PCIe设备的中断传递。所有PCIe设备都必须支持MSI，但软件可能支持也可能不支持MSI，在这种情况下，将使用INTx消息。图17-2还显示了PCIe-to-PCI桥如何将连接PCI设备的边带中断转换为PCIe支持的INTx消息。

---

## 17.2 传统模型

### 17.2.1 概述

为了说明传统中断传递模型，请参考图17-3(第797页)并考虑使用传统中断引脚方法进行中断传递的通常步骤：

![MSI-X](../images/图17-3.MSI-X.png)

*原文图示：MSI-X*


1. **设备置位引脚**：设备通过向控制器置位其引脚来生成中断。在旧系统中，该控制器通常是Intel 8259 PIC，它有15个IRQ输入和一个INTR输出。PIC随后置位INTR以通知CPU有一个或多个中断正在等待处理。

2. **中断确认**：一旦CPU检测到INTR的置位并准备对其采取行动，它必须识别哪个中断实际需要服务，这是通过CPU在处理器总线上发出一个称为**中断确认(Interrupt Acknowledge)**的特殊命令来完成的。

3. **获取中断向量**：该命令由系统路由到PIC，PIC返回一个8位值，称为**中断向量(Interrupt Vector)**，以报告当前正在等待的最高优先级中断。每个IRQ输入的唯一向量将由系统软件预先编程。

4. **索引中断表**：中断处理程序随后使用该向量作为**中断表(Interrupt Table)**(由软件设置的包含所有中断服务程序ISR起始地址的区域)中的偏移量，并获取它在该位置找到的ISR起始地址。

5. **执行中断服务程序**：该地址将指向已设置为处理此中断的ISR的第一条指令。该处理程序将被执行，为中断提供服务并告诉其设备解除置位其INTx#线，然后将控制权返回给先前被中断的任务。

### 17.2.2 支持多处理器的变更

这种模型在单CPU系统中工作良好，但存在一个限制，使其在多CPU系统中不是最优的。问题是INTR引脚只能连接到一个CPU。如果存在多个处理器，那么只有其中一个会看到中断并必须处理所有中断，而其他CPU则看不到任何中断。为了获得最佳性能，这样的系统确实需要系统任务在所有处理器之间均匀分布，称为SMP(Symmetric Multi-Processing，对称多处理)，但引脚模型不支持这一点。

为了实现更好的SMP，需要一种新的模型，为此PIC被修改为IO APIC(Advanced Programmable Interrupt Controller，高级可编程中断控制器)。IO APIC被设计为具有一个单独的小型总线，称为APIC总线，通过它可以传递中断消息，如图17-4所示(第799页)。在这种模型中，消息包含中断向量号，因此CPU不需要向下发送到IO世界来获取它。APIC总线连接到处理器内部称为本地APIC(Local APIC)的新内部逻辑块。该总线在所有代理之间共享，其中任何一个都可以在总线上发起消息，但就我们的目的而言，有趣的部分是外设中断传递的使用。

这些中断现在可以由软件静态分配以由不同的CPU服务，由多个CPU服务，甚至由IO APIC动态分配。

这种称为APIC模型的模型足以使用数年，但仍然依赖于外设的边带引脚才能工作。该模型的另一个限制是进入IO APIC的IRQ数量(中断请求线)。如果没有大量的IRQ，外设必须共享IRQ，这意味着每当该IRQ被置位时都会增加延迟，因为可能有多个设备置位了它，软件必须评估所有这些设备。这种将多个ISR链接在一起的技术通常被称为**中断链(interrupt chaining)**。最终，由于这个问题和其他一些小问题，出现了另一种改进。

为什么不让外设自己直接向本地APIC发送中断消息呢？所需要的只是通信路径，它已经以PCI总线和处理器总线的形式存在。因此APIC总线被消除，所有中断都以存储器写的形式传递到本地APIC，称为MSI或Message Signaled Interrupts(消息信号中断)。这些MSI针对系统理解为中断消息目标的特殊地址(该特殊地址传统上对于基于x86的系统是FEEx_xxxxh)。甚至IO APIC也被编程为使用存储器写(MSI)通过普通数据总线发送其中断通知。现在它只需通过数据总线发送MSI存储器写，目标指向所需处理器的本地APIC的存储器地址，这就达到了通知处理器中断的效果。

这种模型称为xAPIC模型，由于它不是基于进入具有有限输入的中断控制器的边带信号，因此几乎消除了共享中断的需要。有关此模型的更多信息，请参阅"MSI解决方案"(第827页)。

PCI多年前就将MSI支持作为选项添加，而PCIe将该功能设为强制要求。能够自己生成MSI事务的外设为处理中断开辟了新选项，例如让每个功能生成多个唯一中断而不是只有一个的能力。

---

## 17.3 传统PCI中断传递

本节提供有关传统PCI中断传递的更多详细信息。熟悉PCI的读者可能希望继续阅读"虚拟INTx信号"(第805页)以了解PCIe如何模拟此传统模型，或阅读"MSI模型"(第812页)以了解该方法。

使用中断的PCI设备有两个选项。它们可以使用：
- 在原始规范中定义的可以被共享的INTx#低电平有效信号
- 在规范2.2版本中作为选项添加的消息信号中断(MSI)。MSI在PCIe系统中使用无需修改。

### 17.3.1 设备INTx#引脚

PCI设备可以实现最多4个INTx#信号(INTA#、INTB#、INTC#和INTD#)。提供多个引脚是因为PCI设备可以支持最多8个功能，每个功能允许驱动一个(但只能一个)中断引脚。

开发PCI时，典型系统使用包含15输入8259 PIC的芯片组，因此这是系统可用的IRQ数量(映射到中断向量)。然而，其中许多已经用于系统目的，如系统定时器、键盘中断、鼠标中断等。此外，一些引脚保留给仍可插入这些旧系统的ISA卡。因此，PCI规范编写者认为只有四个IRQ可以可靠地用于他们的新总线，因此规范仅支持四个中断引脚。

然而，正如你可能知道的，PCI总线上通常有超过四个PCI设备，甚至单个设备内部可能有超过四个功能，每个功能都希望有自己的中断。这些原因说明了为什么PCI中断被设计为电平敏感和可共享的。这些信号可以简单地通过线或连接在一起，得到少数几个输出，每个代表中断请求。由于它们是共享的，当检测到中断时，中断处理程序软件需要遍历共享同一引脚的功能列表，并测试查看哪些需要服务。

### 17.3.2 确定INTx#引脚支持

PCI功能在其配置头中指示对INTx#信号的支持。只读中断引脚寄存器(Interrupt Pin register)指示此功能是否支持INTx#，如果支持，则在请求中断时将置位哪个中断引脚。

**中断引脚寄存器值：**
- 00h = 未使用INTx#引脚
- 01h = INTA#
- 02h = INTB#
- 03h = INTC#
- 04h = INTD#

### 17.3.3 中断路由

中断线寄存器(Interrupt Line register)提供了驱动程序需要知道的下一个信息：此引脚连接到的PIC的输入引脚。PIC由系统软件为每个输入引脚(IRQ)编程一个唯一的向量号。报告的最高优先级中断 asserted 的向量被报告给处理器，然后处理器使用该向量索引到中断向量表中的相应条目。此条目指向中断设备的中断服务程序，处理器执行该程序。

平台设计人员分配来自设备的INTx#引脚路由。它们可以以多种方式路由，但最终每个INTx#引脚连接到中断控制器的输入。图17-6说明了一个示例，其中几个PCI设备中断通过可编程路由器连接到中断控制器。连接到可编程路由器给定输入的所有信号将被定向到中断控制器的特定输入。中断路由到公共中断控制器输入的功能都将由平台软件(通常是固件)分配相同的中断线号。在此示例中，IRQ15有三个来自不同设备的PCI INTx#输入连接到它。因此，使用这些INTx#线的功能将共享IRQ15，因此都将导致控制器在被查询时发送相同的向量。该向量将使三个不同功能的三个ISR链接在一起。

### 17.3.4 将INTx#线关联到IRQ号

基于系统要求，路由器被编程为将其四个输入连接到四个可用的PIC输入。完成此操作后，与每个功能关联的INTx#引脚的路由已知，中断线号由软件写入每个功能。该值最终由功能的设备驱动程序读取，以便知道它被分配了哪个中断表条目。这是将写入其ISR起始地址的地方，这个过程称为"挂接中断(hooking the interrupt)"。当此功能稍后生成中断时，CPU将接收与中断线寄存器中指定的IRQ对应的中断向量号。CPU使用此向量索引到中断向量表中，以获取与功能的设备驱动程序关联的中断服务程序的入口点。

### 17.3.5 INTx#信号

INTx#线是低电平有效信号，实现为开漏输出，系统在每个线上提供上拉电阻。连接到同一PCI中断请求信号线的多个设备可以同时置位它而不会造成损坏。

当功能信号中断时，它还会设置配置头状态寄存器中的中断状态位(Interrupt Status bit)。系统软件可以读取此位以查看当前是否有中断正在等待。(参见图17-8，第805页)。

**中断禁用(Interrupt Disable)**：PCI 2.3规范在配置头的命令寄存器中添加了中断禁用位(位10)。该位在复位时被清除，允许INTx#信号生成，但软件可以设置它以阻止该信号。请注意，中断禁用位对消息信号中断(MSI)没有影响。MSI通过MSI能力结构中的命令寄存器启用。启用MSI会自动禁用中断引脚或模拟。

**中断状态(Interrupt Status)**：PCI 2.3规范在配置状态寄存器中添加了只读中断状态位(如图17-8所示)。当有待处理的中断时，功能必须设置此状态位。此外，如果配置头命令寄存器中的中断禁用位被清除(即中断启用)，则当设置此状态位时，功能的INTx#信号被置位。此位不受中断禁用位状态的影响。

---

## 17.4 虚拟INTx信号

### 17.4.1 概述

如果环境使得在PCIe拓扑中使用MSI不可能，则将使用INTx信号模型。以下是两个需要使用INTx消息的设备的示例：

**PCIe-to-(PCI或PCI-X)桥** — 大多数PCI设备将使用INTx#引脚，因为MSI支持对它们是可选的。由于PCIe不支持边带中断信号，因此使用带内消息代替。中断控制器理解该消息并将中断请求传递给CPU，其中包括预编程的向量号。

**启动设备(Boot Devices)** — PC系统在启动序列期间通常使用传统中断模型，因为MSI通常需要操作系统级别的初始化。通常，启动需要最少三个子系统：向操作员输出(如视频)、来自操作员的输入(通常是键盘)以及可用于获取操作系统的设备(通常是硬盘)。参与初始化系统的PCIe设备称为"启动设备"。启动设备将使用传统中断支持，直到操作系统和设备驱动程序加载后，此时它们最好使用MSI。

### 17.4.2 虚拟INTx线传递

图17-9(第806页)说明了一个具有PCIe端点和PCI Express-to-PCI桥的系统。如果我们假设软件未在端点上启用MSI，它将使用INTx消息传递中断请求。在此示例中，桥正在使用INTx消息传播来自连接PCI设备的基于引脚的中断。如所见，桥发送INTB消息以信号其来自PCI总线的INTB#输入的置位和解除置位。PCIe端点显示使用模拟消息信号INTA。请注意，INTx#信号涉及两条消息：

- **Assert_INTx消息**：指示虚拟INTx#信号从高到低转换(从非活动到活动)
- **Deassert_INTx消息**：指示从低到高的转换

当功能传递Assert_INTx消息时，它还会设置其配置状态寄存器中的中断状态位，就像它置位物理INTx#引脚一样。

### 17.4.3 INTx消息格式

图17-10(第807页)描述了INTx消息头的格式。中断控制器是这些消息的最终目的地，但使用的路由方法不是"路由到根复合体"，实际上是"本地 - 在接收器终止"，如图17-10所示。有两个原因。第一个是因为每个桥(包括交换机端口和根端口)沿上游路径可能将虚拟中断线映射到桥上的不同虚拟中断线(例如，交换机端口接收Assert_INTA但向上游传播时将其映射为Assert_INTB)。有关此INTx映射的更多信息，请参阅"INTx映射"(第808页)。

这些消息使用本地路由类型的第二个原因是由于我们正在模拟基于引脚的信号。如果端口接收到一个在其主侧映射到INTA的置位中断消息，并且由于先前的中断它已经发送了Assert_INTA消息上游，则没有理由再发送一个。INTA已经被视为已置位。有关此INTx消息折叠的更多信息，请参阅"INTx折叠"(第810页)。

**INTx消息代码：**
- 20h = Assert_INTA
- 21h = Assert_INTB
- 22h = Assert_INTC
- 23h = Assert_INTD
- 24h = Deassert_INTA
- 25h = Deassert_INTB
- 26h = Deassert_INTC
- 27h = Deassert_INTD

### 17.4.4 INTx消息的映射和折叠

**INTx映射**

交换机必须遵守PCI规范定义的INTx映射，如表17-1所示。此映射定义了当中断通过PCI-to-PCI桥路由时存在的虚拟连接。映射基于消息中的请求者ID(Requester ID)字段中的设备号和INTx消息类型。

**表17-1: 虚拟PCI-to-PCI桥上的INTx消息映射**

| 传递INTx的设备号 | INTx消息类型(输入) | INTx消息类型(输出) |
|-----------------|-------------------|-------------------|
| 0, 4, 8, 12等   | INTA              | INTA              |
|                 | INTB              | INTB              |
|                 | INTC              | INTC              |
|                 | INTD              | INTD              |
| 1, 5, 9, 13等   | INTA              | INTB              |
|                 | INTB              | INTC              |
|                 | INTC              | INTD              |
|                 | INTD              | INTA              |
| 2, 6, 10, 14等  | INTA              | INTC              |
|                 | INTB              | INTD              |
|                 | INTC              | INTA              |
|                 | INTD              | INTB              |
| 3, 7, 11, 15等  | INTA              | INTD              |
|                 | INTB              | INTA              |
|                 | INTC              | INTB              |
|                 | INTD              | INTC              |

这种中断映射的原因与PCI相同：尽可能避免多个功能共享相同的INTx#引脚。如前所述，单功能设备如果使用传统中断，则必须使用INTA。因此，如果根端口下游的所有功能都使用INTA，并且桥之间没有映射，它们都将被路由到相同的IRQ。这意味着每当其中一个功能置位INTA时，都必须检查所有功能。这将导致列表末尾功能的显著中断服务延迟。这种中断映射方法是一种粗略的尝试，旨在将中断(特别是INTA)分布到所有四个INTx虚拟线上，因为每个INTx虚拟线都可以在中断控制器映射到单独的IRQ。

**INTx折叠**

PCIe交换机必须确保INTx消息以正确的方式向上游传递。具体来说，必须处理传统PCI实现的中断路由，以便软件可以确定哪些中断被路由到哪些中断控制器输入。INTx#线可以线或连接并路由到中断控制器上的相同IRQ输入，当多个设备在同一条线上信号中断时，中断控制器只看到第一个置位。类似地，当其中一个设备解除置位其INTx#线时，该线保持置位状态直到最后一个被关闭。这些相同的原则适用于PCIe INTx消息。

然而，在某些情况下，两个重叠的INTx消息可能被出口端口的虚拟PCI桥映射到相同的INTx消息，需要折叠这些消息。考虑图17-12(第811页)中说明的以下示例。

当上游交换机端口映射中断消息以在上游链路上传递时，两个中断都将映射为INTB(基于下游交换机端口的设备号)。请注意，因为这两个重叠消息相同，所以必须折叠它们。

折叠确保中断控制器永远不会收到共享中断的两个连续Assert_INTx或Deassert_INTx消息。这相当于INTx信号被线或连接。

### 17.4.5 INTx传递规则

与INTx消息传递相关的规则具有一些独特的特征：

- Assert_INTx和Deassert_INTx仅在上游方向发出
- 正在折叠中断的交换机仅当中断状态发生变化时才会向上游发出INTx消息
- 链路两侧的设备必须跟踪INTA-INTD置位的当前状态
- 交换机跟踪其每个下游端口的四个虚拟线的状态，并可能在其上游端口上呈现折叠的虚拟线集
- 根复合体必须跟踪每个下游端口的四个虚拟线(A-D)的状态
- 可以使用命令寄存器中的中断禁用位禁用INTx信号
- 如果任何INTx虚拟线处于活动状态，然后禁用设备中断，则必须发送相应的Deassert_INTx消息
- 如果下游交换机端口进入DL_Down状态，则必须解除置位任何活动的INTx虚拟线，并相应地更新上游端口(如果该INTx处于活动状态，则需要Deassert_INTx消息)

---

## 17.5 MSI模型

PCIe功能通过MSI能力寄存器指示MSI支持。每个功能必须实现MSI能力结构或MSI-X(eXtended MSI，扩展MSI)能力结构，或两者都实现。MSI能力寄存器由配置软件设置，包括：
- 目标存储器地址
- 要写入该地址的数据值
- 可以编码到数据中的唯一消息数量

注意：MSI始终具有1DW的数据负载。

### 17.5.1 MSI能力结构

MSI能力结构位于PCI兼容配置空间区域(前256字节)中。根据它是支持64位寻址还是仅支持32位，以及是否支持每个向量的屏蔽，MSI能力结构有四种变体。原生PCIe设备需要支持64位寻址。MSI能力结构的所有四种变体如图17-13所示(第813页)。

**MSI能力结构变体：**

1. **32位地址**
   - DW0: Capability ID (05h), Next Capability Pointer, Message Control
   - DW1: Message Address [31:0]
   - DW2: Message Data

2. **64位地址**
   - DW0: Capability ID (05h), Next Capability Pointer, Message Control
   - DW1: Message Address [31:0]
   - DW2: Message Address [63:32]
   - DW3: Message Data

3. **32位地址带每个向量屏蔽**
   - DW0: Capability ID (05h), Next Capability Pointer, Message Control
   - DW1: Message Address [31:0], Reserved
   - DW2: Message Data
   - DW3: Mask Bits
   - DW4: Pending Bits

4. **64位地址带每个向量屏蔽**
   - DW0: Capability ID (05h), Next Capability Pointer, Message Control
   - DW1: Message Address [31:0]
   - DW2: Message Address [63:32], Reserved
   - DW3: Message Data
   - DW4: Mask Bits
   - DW5: Pending Bits

**能力ID(Capability ID)**：能力ID值05h标识MSI能力，是只读值。

**下一个能力指针(Next Capability Pointer)**：寄存器的第二个字节是只读值，给出从配置空间顶部到链表中下一个能力结构的dword对齐偏移量，或者包含00h以指示链表结束。

**消息控制寄存器(Message Control Register)**：

| 位   | 字段名                    | 描述 |
|-----|--------------------------|------|
| 0   | MSI Enable               | 读/写。复位后的状态为0，表示设备的MSI能力被禁用。0 = 功能被禁用使用MSI，必须使用MSI-X或INTx消息。1 = 功能被启用使用MSI请求服务，不会使用MSI-X或INTx消息。 |
| 3:1 | Multiple Message Capable | 只读。系统软件读取此字段以确定功能希望使用多少消息(中断向量)。请求的消息数量是2的幂，因此希望三个消息的功能必须请求分配四个消息给它。 |
| 6:4 | Multiple Message Enable  | 读/写。系统软件读取Multiple Message Capable字段后，在此字段中编程一个3位值，指示分配给功能的实际消息数量。分配的数量可以等于或小于实际请求的数量。复位后此字段的状态为000b。 |
| 7   | 64-bit Address Capable   | 只读。0 = 功能不实现消息地址寄存器的高32位；只有32位地址可能。1 = 功能实现消息地址寄存器的高32位，能够生成64位存储器地址。 |
| 8   | Per-Vector Masking Capable | 只读。0 = 功能不实现屏蔽位寄存器或待定位寄存器；软件没有能力使用此能力结构屏蔽单个中断。1 = 功能实现屏蔽位寄存器或待定位寄存器；软件有能力使用此能力结构屏蔽单个中断。 |

**消息地址寄存器(Message Address Register)**：32位消息地址寄存器的低两位为零且无法更改，强制软件分配的地址为dword对齐。通常，这将是系统CPU中本地APIC的地址。在基于x86的系统(英特尔兼容)中，此地址传统上为FEEx_xxxxh，其中低20位指示正在目标的本地APIC以及有关中断本身的一些其他信息。重要的是要注意如何解释地址是平台特定的，不在PCI或PCIe规范中规定。

包含位[63:32]的寄存器对于原生PCI Express设备是必需的，但对于传统端点是可选的。如果消息控制寄存器的位7被设置，则存在此寄存器。如果是这样，它是一个读/写寄存器，与消息地址[31:0]寄存器一起使用，以实现64位存储器地址用于来自此功能的中断传递。

**消息数据寄存器(Message Data Register)**：系统软件将基本消息数据模式写入此16位读/写寄存器。当功能生成中断请求时，它将32位数据值写入MSI能力寄存器中地址字段指定的存储器地址。此数据的高16位始终设置为零，而低16位由消息数据寄存器提供。

如果为功能分配了多个消息，它通过修改分配的消息数据值的低位(可修改的位数取决于配置软件分配给功能的消息数量)来形成希望报告的事件的适当值。

**屏蔽位寄存器和待定位寄存器(Mask Bits Register and Pending Bits Register)**：如果功能支持每个向量的屏蔽(在消息控制寄存器的位[8]中指示)，则存在这些寄存器。使用MSI的功能可以请求和分配的最大中断消息(中断向量)数量为32。因此，这两个寄存器长度为32位，每个潜在中断消息都有自己的屏蔽位和待定位。如果屏蔽位寄存器的位[0]被设置，则中断消息0被屏蔽(这是来自此功能的基本向量)。如果位[1]被设置，则中断消息1被屏蔽(这是基本向量+1)。

当屏蔽中断消息时，该向量的MSI无法发送。相反，设置相应的待定位。这允许软件屏蔽来自功能的单个中断，然后定期轮询功能以查看是否有任何被屏蔽的中断正在等待。

如果软件清除屏蔽位且相应的待定位被设置，则功能必须在此时发送MSI请求。一旦发送了中断消息，功能将清除待定位。

### 17.5.2 MSI配置基础

以下列表指定了软件为PCI Express设备配置MSI中断所采取的步骤：

1. 在启动时，枚举软件扫描系统中的所有PCI兼容功能。
2. 一旦发现功能，软件读取能力列表指针，以找到链表中第一个能力结构的位置。
3. 如果在列表中找到MSI能力结构(能力ID为05h)，软件读取设备消息控制寄存器中的Multiple Message Capable字段，以确定设备支持多少事件特定消息，以及它是否支持64位消息地址或仅支持32位。然后软件分配等于或少于此数量的消息数量，并将该值写入Multiple Message Enable字段。至少，一个消息将分配给设备。
4. 软件将基本消息数据模式写入设备的消息数据寄存器，并将dword对齐的存储器地址写入设备的消息地址寄存器，作为MSI写的目标地址。
5. 最后，软件设置设备消息控制寄存器中的MSI Enable位，使其能够生成MSI写并禁用其他中断传递选项。

### 17.5.3 生成MSI中断请求的基础

MSI存储器写事务头和数据字段的内容的关键点包括：
- 格式字段对于原生功能必须为011b，表示具有数据的4DW头(64位地址)，但对于传统端点可能为010b，表示32位地址。
- No Snoop和Relaxed Ordering的属性位必须为零。
- 长度字段必须为01h，表示最大数据负载为1DW。
- First BE字段必须为1111b，表示DW的所有四个字节都有有效数据，即使MSI的上两个字节始终为零。
- Last BE字段必须为0000b，表示单DW事务。
- 头内的地址字段直接来自MSI能力寄存器内的地址字段。
- 数据负载的低16位来自MSI能力寄存器内的数据字段。

### 17.5.4 多条消息

如果系统软件为功能分配了多个消息，则通过修改分配的消息数据值的低位来创建多个值，以为每个设备特定事件类型发送不同的消息。

例如，假设：
- 四个消息已分配给设备
- 值49A0h已分配给设备的消息数据寄存器
- 存储器地址FEEF_F00Ch已写入设备的消息地址寄存器
- 当四个事件之一发生时，设备通过执行对存储器地址FEEF_F00Ch的dword写来生成请求，数据值为0000_49A0h、0000_49A1h、0000_49A2h或0000_49A3h。换句话说，修改数据值的低两位以指定发生了哪个事件。如果为此功能分配了8个消息，则可以修改低三位。此外，设备始终使用0000h作为其消息数据值的上2个字节。

---

## 17.6 MSI-X模型

### 17.6.1 概述

PCI规范的3.0版本添加了对MSI-X的支持，它有自己的能力结构。MSI-X的动机是希望缓解MSI的三个缺点：
- 每个功能32个向量对某些应用来说不够
- 只有一个目标地址使得中断在多个CPU之间的静态分布变得困难。如果可以为每个向量分配唯一地址，将实现最大的灵活性。
- 在多个平台中，如基于x86的系统，中断的向量号指示其相对于其他中断的优先级。使用MSI，单个功能可以被分配多个中断，但所有中断向量将是连续的，意味着类似的优先级。如果此功能的一些中断应该是高优先级而另一些应该是低优先级，这不是一个好的解决方案。更好的方法是让软件为分配给功能的每个中断指定唯一的向量(消息数据值)，该向量不需要是连续的。

牢记这些目标，很容易理解为实现更多向量而实施的寄存器更改，每个向量被分配目标地址和消息数据值。

### 17.6.2 MSI-X能力结构

如图17-17所示，消息控制寄存器与MSI有很大不同。有趣的是，尽管MSI-X每个功能可以支持多达2048个向量，而MSI只有32个，但MSI-X的配置寄存器数量实际上比MSI少一些。这是因为向量信息不包含在这里。相反，它在由表BIR(基地址指示寄存器)指向的存储器位置(MMIO)中，如图17-18所示。

**MSI-X能力结构：**
- DW0: Capability ID (11h), Next Capability Pointer, Message Control
- DW1: Table BIR, MSI-X Table Offset
- DW2: PBA BIR, Pending Bit Array Offset

**MSI-X消息控制寄存器格式：**

| 位    | 字段名        | 描述 |
|------|--------------|------|
| 10:0 | Table Size   | 只读。此字段指示此功能支持的中断消息(向量)数量。此处的值以N-1方式解释，因此值0表示1个向量。值7表示8个向量。每个向量在MSI-X表中都有自己的条目，在待定位数组中都有自己的位。 |
| 13:11| Reserved     | 只读。始终为零。 |
| 14   | Function Mask| 读/写。此字段为系统软件提供了一种简单的方法来屏蔽来自功能的所有中断。如果此位被清除，仍然可以通过设置每个向量的MSI-X表条目中的屏蔽位来单独屏蔽中断。 |
| 15   | MSI-X Enable | 读/写。复位后的状态为0，表示设备的MSI-X能力被禁用。0 = 功能被禁用使用MSI-X，必须使用MSI或INTx消息。1 = 功能被启用使用MSI-X请求服务，不会使用MSI或INTx消息。 |

### 17.6.3 MSI-X表

MSI-X表本身是一个向量和地址数组，如图17-19所示。每个条目代表一个向量，包含四个Dword。DW0和DW1为该向量提供唯一的64位地址，而DW2为其提供唯一的32位数据模式。DW3目前只包含一位：该向量的屏蔽位，允许根据需要独立屏蔽每个向量。

**MSI-X表条目：**
- DW0: Lower Address (低地址)
- DW1: Upper Address (高地址)
- DW2: Message Data (消息数据)
- DW3: Vector Control (向量控制，位0是向量屏蔽位R/W)

### 17.6.4 待定位数组(Pending Bit Array)

与MSI-X表类似，待定位数组也位于存储器地址内。它可以使用与MSI-X表相同的BIR值(相同的BAR)但偏移量不同，或者可以使用完全不同的BAR。该数组如图17-20所示，只包含将使用的每个向量的位。如果触发该中断的事件发生但其屏蔽位已被设置，则不会发送MSI-X事务。相反，设置相应的待定位。稍后，如果该向量被取消屏蔽且待定位仍被设置，则将在此时生成中断。

---

## 17.7 进入中断处理程序时的存储器同步

### 17.7.1 问题

任何中断方案在传递数据时都存在潜在问题。例如，如果设备先前已发送数据并希望通过中断报告，数据传递的意外延迟可能允许中断过早到达。这可能发生在图17-21(第827页)所示的桥数据缓冲区中，结果是竞争条件。步骤与我们之前的讨论类似：

1. 功能向存储器写入数据块。写在本地总线上作为posted事务完成，意味着发送者已完成其需要做的所有事情，事务被视为已完成。
2. 传递中断以通知软件某些请求的数据现在存在于存储器中。然而，数据由于某种原因在桥中被延迟。
3. 像以前一样获取中断向量。
4. 获取ISR起始地址并将控制权传递给它。
5. ISR从目标存储器缓冲区读取，但数据负载仍未被传递，因此它获取陈旧数据，可能导致错误。

### 17.7.2 一种解决方案

缓解此问题的一种方法是利用PCI事务排序规则。如果ISR在尝试获取数据之前首先向发起中断的设备发送读请求，则产生的读完成将遵循从该设备到存储器的任何写数据所采用的相同路径返回到CPU。事务排序规则保证桥中的读结果不能通过同方向的posted写，因此最终结果是数据将被写入存储器，然后读结果才被允许到达CPU。因此，如果ISR等待读完成到达后再继续，它可以确保任何数据都已被传递到存储器，从而避免竞争条件。由于读基本上被用作数据刷新机制，因此不需要它返回任何数据。在这种情况下，读可以为零长度，返回的数据被丢弃。因此，这种类型的读有时称为"虚拟读(dummy read)"。

### 17.7.3 MSI解决方案

MSI可以简化此过程，尽管有一些要求才能使其工作。如果系统允许设备生成自己的MSI写而不是通过IO APIC等中介，则可以发生以下示例：

1. 设备向存储器写入有效负载数据，它被桥中的写缓冲区吸收。
2. 设备认为数据已被传递并信号中断以通知CPU。在这种情况下，发送MSI并使用与数据相同的路径。由于数据和MSI对桥都显示为存储器写，正常的事务排序规则将保持它们的正确顺序。
3. 有效负载数据被传递到存储器，为MSI写释放通过桥的路径。
4. MSI写被传递到CPU本地APIC，软件现在知道有效负载数据可用。

### 17.7.4 流量类别必须匹配

然而，这里必须强调一个重要点。数据和MSI必须使用相同的流量类别(Traffic Class)才能使其工作。回想一下，已被分配不同TC值的数据包可能最终被映射到不同的虚拟通道中，而不同VC中的数据包没有排序关系。如果数据被映射到VC0而MSI被映射到VC1，则系统将不知道它们之间的任何排序关系，无法自动强制执行存储器一致性。

如果无法为两个数据包分配相同的TC，则系统将需要使用"虚拟读"方法，读请求的TC需要与数据写数据包的TC匹配。应该清楚的是，即使两者使用相同的TC，也必须避免使用Relaxed Ordering位。我们依靠事务排序规则来实现存储器同步，因此它们不能被放宽。

---

## 17.8 中断延迟

中断延迟是从设备决定需要中断到CPU开始执行相应中断服务程序之间的时间。影响中断延迟的因素包括：

1. **链路状态**：如果链路处于低功耗状态(L0s或L1)，需要额外的时间来退出这些状态。
2. **仲裁延迟**：MSI写事务可能需要等待仲裁访问链路。
3. **系统拓扑深度**：在到达根复合体之前通过多个交换机级别会增加延迟。
4. **软件开销**：操作系统处理中断并调度ISR所需的时间。

MSI通常比传统INTx方法具有更低的中断延迟，因为：
- 不需要中断确认周期
- 不需要软件读取中断控制器以确定中断源
- 消息直接传递到目标CPU的本地APIC

---

## 17.9 MSI可能导致错误

虽然MSI提供了许多优势，但也可能引入错误情况：

1. **地址错误**：如果MSI目标地址配置不正确，中断写可能到达错误的存储器位置。
2. **数据损坏**：MSI写数据值中的错误可能导致CPU执行错误的中断向量。
3. **排序违规**：如果MSI和数据不使用相同的流量类别，可能导致存储器一致性违规。
4. **超时**：如果MSI写由于链路错误而无法传递，设备可能无法通知CPU需要服务。

---

## 17.10 一些MSI规则和建议

1. **所有PCIe设备必须支持MSI**：这是PCIe规范的基本要求。
2. **软件应优先使用MSI而非INTx**：MSI提供更好的性能和可扩展性。
3. **多个消息应分配给高带宽设备**：这允许不同类型的中断事件使用不同的向量。
4. **MSI目标地址应对齐到4字节边界**：确保正确的存储器访问。
5. **避免混合使用MSI和MSI-X**：一个功能应使用其中一种，而不是同时使用两者。
6. **正确配置流量类别**：确保MSI和数据事务使用相同的TC以保持排序。

---

## 17.11 基本系统外设的特殊考虑

基本系统外设(如启动设备)可能需要特殊的中断处理考虑：

1. **启动期间的传统中断**：在操作系统加载之前，设备可能需要使用INTx消息。
2. **固件支持**：系统固件必须正确配置中断路由以支持传统模式。
3. **驱动程序转换**：设备驱动程序应在初始化期间从INTx模式转换到MSI模式。
4. **回退机制**：如果MSI配置失败，设备应能够回退到INTx模式。

### 传统系统示例

在传统系统中，中断处理可能涉及以下组件：
- **8259 PIC**：用于单处理器系统的传统中断控制器。
- **IO APIC**：用于多处理器系统的高级中断控制器。
- **本地APIC**：集成在CPU中的中断接收逻辑。
- **中断路由器**：平台特定的逻辑，用于将PCI INTx#信号映射到IRQ。

系统软件必须正确配置这些组件以确保中断正确传递。对于PCIe设备，这意味着：
1. 在枚举期间检测设备的MSI能力
2. 分配适当数量的消息向量
3. 配置目标地址(通常是本地APIC地址)
4. 启用MSI并禁用INTx模拟

---

## 总结

本章介绍了PCIe中中断支持的三种主要机制：

1. **传统INTx模拟**：通过消息模拟PCI INTx#引脚行为，用于向后兼容。
2. **MSI(Message Signaled Interrupts)**：使用存储器写传递中断，所有PCIe设备必须支持。
3. **MSI-X**：MSI的扩展版本，支持更多向量(最多2048个)和每个向量的独立目标地址。

MSI和MSI-X相对于传统INTx方法的优势包括：
- 消除边带信号，减少引脚数量
- 支持更多的中断向量
- 更好的多处理器支持
- 更低的中断延迟
- 避免中断共享

系统软件负责在设备枚举期间配置中断机制，选择最适合设备需求的方法。正确的中断配置对于系统性能和可靠性至关重要。

---

## 17.12 Linux内核MSI实现

### 源码位置
`drivers/pci/msi.c`

### MSI使能控制
```c
static int pci_msi_enable = 1;  // 全局MSI使能开关
```

### 关键数据结构

1. **MSI描述符**
   ```c
   struct msi_desc {
       struct list_head list;
       struct msi_msg msg;           // MSI消息（地址+数据）
       struct irq_domain *domain;
       unsigned int irq;             // Linux IRQ号
       unsigned int nvec_used;       // 使用的向量数
       unsigned int mask_pos;        // 屏蔽寄存器位置
       u32 masked;                   // 当前屏蔽值
       // ...
   };
   ```

2. **MSI消息格式**
   ```c
   struct msi_msg {
       u32 address_lo;               // 消息地址低32位
       u32 address_hi;               // 消息地址高32位（64位支持）
       u32 data;                     // 消息数据
       u32 device_addr;              // 设备地址
   };
   ```

### 关键接口

1. **分配MSI中断**
   ```c
   int pci_alloc_irq_vectors(struct pci_dev *dev, unsigned int min_vecs,
                            unsigned int max_vecs, unsigned int flags);
   ```

2. **释放MSI中断**
   ```c
   void pci_free_irq_vectors(struct pci_dev *dev);
   ```

3. **MSI屏蔽/取消屏蔽**
   ```c
   void __pci_msi_desc_mask_irq(struct msi_desc *desc, u32 mask, u32 flag);
   ```

### MSI-X支持

```c
// MSI-X表大小计算
#define msix_table_size(flags) ((flags & PCI_MSIX_FLAGS_QSIZE) + 1)

// MSI-X表条目（4个DWORD）
// DW0: Lower Address
// DW1: Upper Address
// DW2: Message Data
// DW3: Vector Control（位0为屏蔽位）
```

### 中断分配策略

Linux内核自动选择最优中断机制：
1. 优先尝试MSI-X（如果设备支持且系统可用）
2. 其次尝试MSI
3. 最后回退到INTx模拟

### 用户接口

```bash
# 查看设备MSI信息
lspci -vvv -s 0000:01:00.0 | grep -i msi

# 查看中断统计
cat /proc/interrupts | grep -i msi
```

---

**术语对照表**

| 英文术语 | 中文翻译 |
|---------|---------|
| Interrupt | 中断 |
| Message Signaled Interrupt (MSI) | 消息信号中断 |
| MSI-X | 扩展消息信号中断 |
| INTx | 传统中断(INTA#/INTB#/INTC#/INTD#) |
| Root Complex | 根复合体 |
| Endpoint | 端点 |
| Switch | 交换机 |
| Bridge | 桥 |
| Local APIC | 本地APIC |
| IO APIC | IO APIC |
| Interrupt Vector | 中断向量 |
| Interrupt Service Routine (ISR) | 中断服务程序 |
| Posted Memory Write | 发布的存储器写 |
| Traffic Class (TC) | 流量类别 |
| Virtual Channel (VC) | 虚拟通道 |

---

## 17.10 Linux内核实现参考（平台特定补充）

### 17.10.1 飞腾PCIe EP的MSI实现

飞腾PCIe EP控制器实现了MSI中断发送功能，与本书第17章介绍的MSI机制完全对应。

#### MSI能力寄存器

```c
// 来自 drivers/pci/controller/pcie-phytium-register.h
#define PHYTIUM_PCI_INTERRUPT_PIN       0xa8
#define  INTERRUPT_PIN_MASK             0x7
#define  MSI_DISABLE                    (1 << 3)   // MSI禁用
#define  MSI_NUM_MASK                   (0x7)      // MSI数量掩码
#define  MSI_NUM_SHIFT                  4          // MSI数量移位
#define  MSI_MASK_SUPPORT               (1 << 7)   // MSI掩码支持
#define PHYTIUM_PCI_CF_MSI_BASE         0x10e0     // MSI能力基址
#define PHYTIUM_PCI_CF_MSI_CONTROL      0x10e2     // MSI控制寄存器
```

**与书中17.5节的对应**：

书中MSI能力结构：
```
MSI Capability ID
Message Control (MMC, MME, Enable)
Message Address (32/64-bit)
Message Data
```

飞腾实现：
- `MSI_NUM_MASK/MSI_NUM_SHIFT` ←→ 书中Message Control的MMC字段
- `MSI_MASK_SUPPORT` ←→ 书中Mask/Pending支持
- `PHYTIUM_PCI_CF_MSI_BASE` ←→ 书中MSI能力结构基址

#### MSI配置代码

```c
// 来自 drivers/pci/controller/pcie-phytium-ep.c
static int phytium_pcie_ep_set_msi(struct pci_epc *epc, u8 fn, u8 mmc)
{
    struct phytium_pcie_ep *priv = epc_get_drvdata(epc);
    u16 flags = 0;
    
    // 设置MSI向量数量（对应书中17.5.1节Message Control）
    flags = (mmc & MSI_NUM_MASK) << MSI_NUM_SHIFT;
    
    // 禁用MSI掩码支持
    flags &= ~MSI_MASK_SUPPORT;
    
    phytium_pcie_writew(priv, fn, PHYTIUM_PCI_INTERRUPT_PIN, flags);
    return 0;
}

// 读取MSI配置
static int phytium_pcie_ep_get_msi(struct pci_epc *epc, u8 fn)
{
    struct phytium_pcie_ep *priv = epc_get_drvdata(epc);
    u16 flags, mme;
    u32 cap = PHYTIUM_PCI_CF_MSI_BASE;
    
    // 读取MSI标志（对应书中17.5.1节）
    flags = phytium_pcie_readw(priv, fn, cap + PCI_MSI_FLAGS);
    if (!(flags & PCI_MSI_FLAGS_ENABLE))
        return -EINVAL;
    
    // 获取MSI向量数量（对应书中MME字段）
    mme = (flags & PCI_MSI_FLAGS_QSIZE) >> 4;
    
    return mme;
}
```

#### MSI发送实现

```c
// MSI中断发送（对应书中17.5.3节MSI发送）
static int phytium_pcie_ep_send_msi_irq(struct phytium_pcie_ep *priv, u8 fn,
                                        u8 interrupt_num)
{
    u32 cap = PHYTIUM_PCI_CF_MSI_BASE;
    u16 flags, mme, data_mask, data;
    u8 msi_count;
    u64 pci_addr;
    u32 src_addr0, src_addr1, trsl_addr0, trsl_addr1, trsl_param, atr_size;
    
    // 读取MSI配置（对应书中17.5.1节）
    flags = phytium_pcie_readw(priv, fn, cap + PCI_MSI_FLAGS);
    if (!(flags & PCI_MSI_FLAGS_ENABLE))
        return -EINVAL;
    
    // 获取MSI向量数量（对应书中MME字段）
    mme = (flags & PCI_MSI_FLAGS_QSIZE) >> 4;
    msi_count = 1 << mme;
    if (!interrupt_num || interrupt_num > msi_count)
        return -EINVAL;
    
    // 构造MSI数据（对应书中17.5.1节Message Data）
    data_mask = msi_count - 1;
    data = phytium_pcie_readw(priv, fn, cap + PCI_MSI_DATA_64);
    data = (data & ~data_mask) | ((interrupt_num - 1) & data_mask);
    
    // 获取MSI地址（对应书中17.5.1节Message Address）
    pci_addr = phytium_pcie_readl(priv, fn, cap + PCI_MSI_ADDRESS_HI);
    pci_addr <<= 32;
    pci_addr |= phytium_pcie_readl(priv, fn, cap + PCI_MSI_ADDRESS_LO);
    pci_addr &= GENMASK_ULL(63, 2);
    
    // 配置地址转换并发送MSI
    // MSI通过Memory Write TLP发送（对应书中17.5.3节）
    atr_size = fls64(pci_addr_mask) - 1;
    src_addr0 = ATR_IMPL | ((atr_size & ATR_SIZE_MASK) << ATR_SIZE_SHIFT);
    src_addr0 |= (lower_32_bits(priv->irq_phys_addr) & SRC_ADDR_32_12_MASK);
    src_addr1 = upper_32_bits(priv->irq_phys_addr);
    trsl_addr0 = (lower_32_bits(pci_addr) & TRSL_ADDR_32_12_MASK);
    trsl_addr1 = upper_32_bits(pci_addr);
    trsl_param = TRSL_ID_PCIE_TR;
    
    // 写入地址转换寄存器
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_SLAVE0_SRC_ADDR0(0), src_addr0);
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_SLAVE0_SRC_ADDR1(0), src_addr1);
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_SLAVE0_TRSL_ADDR0(0), trsl_addr0);
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_SLAVE0_TRSL_ADDR1(0), trsl_addr1);
    phytium_pcie_writel(priv, fn, PHYTIUM_PCI_SLAVE0_TRSL_PARAM(0), trsl_param);
    
    return 0;
}
```

**与书中17.5.3节的对应**：

书中MSI发送过程：
1. 设备发送Memory Write TLP到MSI地址
2. 数据字段包含中断向量号
3. 使用特定的TC和VC

飞腾实现：
1. 配置地址转换（ATR）将MSI地址映射到物理地址
2. 通过`TRSL_ID_PCIE_TR`参数发送PCIe事务
3. 数据字段构造与书中一致

### 17.10.2 Linux内核MSI框架

Linux内核的MSI/MSI-X实现与书中介绍的中断机制对应：

```c
// 来自 drivers/pci/msi.c
struct msi_desc {
    struct list_head list;
    struct msi_msg msg;           // MSI消息（对应书中Message Address/Data）
    struct irq_affinity_desc *affinity;
    unsigned int irq;             // Linux IRQ号
    unsigned int nvec_used;       // 使用的向量数
    // ...
};

// MSI使能（对应书中17.5.2节MSI配置）
int pci_enable_msi(struct pci_dev *dev)
{
    // ...
    // 分配MSI描述符
    // 配置MSI能力寄存器
    // 注册中断处理程序
}

// MSI-X使能（对应书中17.6节MSI-X）
int pci_enable_msix_range(struct pci_dev *dev, struct msix_entry *entries,
                          int minvec, int maxvec)
{
    // ...
    // 分配MSI-X表
    // 配置每个向量的地址和数据
}
```

### 17.10.3 实际应用建议

**MSI调试技巧**：
1. 使用`cat /proc/interrupts`查看MSI中断统计
2. 检查MSI能力寄存器配置（`lspci -vv`）
3. 验证MSI地址和数据设置

**性能优化**：
- 使用MSI-X替代MSI以支持更多中断向量（对应17.6节）
- 合理配置中断亲和性（Affinity）
- 避免共享中断向量

**故障排查**：
- MSI无法触发：检查MSI Enable位
- 中断丢失：检查Mask/Pending位
- 地址错误：验证Message Address配置

---

*翻译完成于: 2026-03-17*
*原文来源: MindShare PCI Express Technology 3.0, Chapter 17*
*平台补充: 飞腾PCIe EP MSI实现参考*


---

## 本章图片附录

以下是本章相关的原文图片：


### 图17-1.INTx.png

![图17-1.INTx.png](../images/图17-1.INTx.png)


### 图17-2.MSI.png

![图17-2.MSI.png](../images/图17-2.MSI.png)


### 图17-3.MSI-X.png

![图17-3.MSI-X.png](../images/图17-3.MSI-X.png)

