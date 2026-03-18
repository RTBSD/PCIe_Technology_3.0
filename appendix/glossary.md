# PCIe Technology 3.0 翻译术语表

## 说明

本术语表基于PCIe Skill手册和PCI Express规范制定，用于确保翻译的一致性。

---

## 核心术语

### 架构组件

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Root Complex | 根复合体 | RC | 也可译为"根联合体" |
| Endpoint | 端点 | EP | |
| Switch | 交换机 | | 也可译为"交换开关" |
| Bridge | 桥 | | PCI-to-PCI Bridge |
| Lane | 通道/链路 | | 根据上下文选择 |
| Link | 链路 | | 由1-32个Lane组成 |
| Port | 端口 | | |
| Upstream | 上行 | | 指向Root Complex方向 |
| Downstream | 下行 | | 远离Root Complex方向 |

### 协议层

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Transaction Layer | 事务层 | TL | |
| Data Link Layer | 数据链路层 | DLL | |
| Physical Layer | 物理层 | PHY | |
| TLP | 事务层包 | TLP | Transaction Layer Packet |
| DLLP | 数据链路层包 | DLLP | Data Link Layer Packet |
| Ordered Set | 有序集 | | |

### 事务类型

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Memory Read | 存储器读 | MRd | |
| Memory Write | 存储器写 | MWr | |
| Configuration Read | 配置读 | CfgRd | |
| Configuration Write | 配置写 | CfgWr | |
| I/O Read | I/O读 | IORd | |
| I/O Write | I/O写 | IOWr | |
| Message | 消息 | Msg | |
| Completion | 完成 | Cpl | |
| Posted Transaction | 发布事务 | | 非报告式事务 |
| Non-Posted Transaction | 非发布事务 | | 报告式事务 |

### 配置与地址

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Configuration Space | 配置空间 | | |
| Base Address Register | 基地址寄存器 | BAR | |
| Memory Space | 存储器空间 | | |
| I/O Space | I/O空间 | | |
| Prefetchable Memory | 可预取存储器 | | |
| Bus Number | 总线号 | | |
| Device Number | 设备号 | | |
| Function Number | 功能号 | | |
| Enumeration | 枚举 | | 拓扑发现过程 |

### 流量控制

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Flow Control | 流量控制 | FC | |
| Virtual Channel | 虚拟通道 | VC | |
| Traffic Class | 流量类别 | TC | |
| Credit | 信用量/流控信用 | | |
| Credit-Based Flow Control | 基于信用量的流量控制 | | |
| Initial Flow Control | 初始流量控制 | | |
| Update Flow Control | 更新流量控制 | | |

### 服务质量

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Quality of Service | 服务质量 | QoS | |
| Arbitration | 仲裁 | | |
| Port Arbitration | 端口仲裁 | | |
| VC Arbitration | 虚拟通道仲裁 | | |
| Isochronous | 等时/同步 | | 时间敏感数据传输 |
| Latency | 延迟 | | |
| Bandwidth | 带宽 | | |

### 事务排序

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Transaction Ordering | 事务排序 | | |
| Ordering Rules | 排序规则 | | |
| Relaxed Ordering | 宽松排序 | RO | |
| Weak Ordering | 弱排序 | | |
| ID-Based Ordering | 基于ID的排序 | IDO | |
| Producer/Consumer Model | 生产者/消费者模型 | | |
| Deadlock Avoidance | 死锁避免 | | |

### Ack/Nak协议

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Ack/Nak Protocol | 确认/否定确认协议 | | |
| Acknowledge | 确认 | Ack | |
| Negative Acknowledge | 否定确认 | Nak | |
| Sequence Number | 序列号 | | |
| LCRC | 链路循环冗余校验 | LCRC | Link CRC |
| ECRC | 端到端循环冗余校验 | ECRC | End-to-End CRC |
| Replay Buffer | 重放缓冲区 | | |
| Replay Timer | 重放定时器 | | |

### 物理层

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| 8b/10b Encoding | 8b/10b编码 | | Gen1/Gen2 |
| 128b/130b Encoding | 128b/130b编码 | | Gen3 |
| Scrambling | 加扰 | | |
| Descrambling | 解扰 | | |
| Serializer | 串行化器 | | |
| Deserializer | 解串行化器 | | |
| Differential Signaling | 差分信号 | | |
| Clock Recovery | 时钟恢复 | | |
| Bit Lock | 位锁定 | | |
| Symbol Lock | 符号锁定 | | |
| Block Alignment | 块对齐 | | Gen3 |
| Elastic Buffer | 弹性缓冲区 | | |
| Lane-to-Lane Skew | 通道间偏斜 | | |
| De-skew | 去偏斜 | | |

### 链路训练

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Link Training | 链路训练 | | |
| Link Initialization | 链路初始化 | | |
| LTSSM | 链路训练与状态状态机 | LTSSM | Link Training and Status State Machine |
| Training Sequence | 训练序列 | TS | TS1/TS2 Ordered Set |
| Electrical Idle | 电气空闲 | | |
| Recovery | 恢复 | | |
| Polling | 轮询 | | |
| Configuration | 配置 | | |
| L0 | L0状态 | | 正常工作状态 |
| L0s | L0s状态 | | 低功耗状态 |
| L1 | L1状态 | | 更低功耗状态 |
| L2 | L2状态 | | 最低功耗状态 |

### 电气特性

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Equalization | 均衡 | | |
| Transmitter Equalization | 发送端均衡 | Tx EQ | |
| Receiver Equalization | 接收端均衡 | Rx EQ | |
| De-emphasis | 去加重 | | |
| Pre-shoot | 预冲 | | |
| Boost | 提升 | | |
| Eye Diagram | 眼图 | | |
| Jitter | 抖动 | | |
| Signal Integrity | 信号完整性 | SI | |
| Spread Spectrum Clocking | 扩频时钟 | SSC | |

### 错误处理

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Error Detection | 错误检测 | | |
| Error Handling | 错误处理 | | |
| Correctable Error | 可纠正错误 | | |
| Uncorrectable Error | 不可纠正错误 | | |
| Fatal Error | 致命错误 | | |
| Non-Fatal Error | 非致命错误 | | |
| Advanced Error Reporting | 高级错误报告 | AER | |
| Error Pollution | 错误污染 | | |

### 电源管理

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Power Management | 电源管理 | PM | |
| Active State Power Management | 主动状态电源管理 | ASPM | |
| Device Power State | 设备电源状态 | | D0-D3 |
| Link Power State | 链路电源状态 | | L0-L3 |
| Power Management Event | 电源管理事件 | PME | |

### 中断

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Interrupt | 中断 | | |
| Legacy Interrupt | 传统中断 | | INTx |
| Message Signaled Interrupt | 消息信号中断 | MSI | |
| MSI-X | 扩展消息信号中断 | MSI-X | |
| INTx | 传统中断信号 | | Interrupt x |

### 复位

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Reset | 复位 | | |
| Fundamental Reset | 基本复位 | | |
| Hot Reset | 热复位 | | |
| Function Level Reset | 功能级复位 | FLR | |
| PERST# | 外设复位信号 | | |

### 热插拔

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Hot Plug | 热插拔 | | |
| Surprise Removal | 意外移除 | | |
| Attention Indicator | 注意指示灯 | | |
| Power Indicator | 电源指示灯 | | |
| Electromechanical Interlock | 机电互锁 | | |

### 速率与代际

| 英文术语 | 中文翻译 | 缩写 | 备注 |
|----------|----------|------|------|
| Generation 1 | 第一代 | Gen1 | 2.5 GT/s |
| Generation 2 | 第二代 | Gen2 | 5.0 GT/s |
| Generation 3 | 第三代 | Gen3 | 8.0 GT/s |
| GT/s | 千兆传输每秒 | | GigaTransfers per second |

---

## 翻译规范

### 1. 首次出现原则
技术术语首次出现时，采用以下格式：
```
Transaction Layer Packet (TLP, 事务层包)
```

### 2. 缩写保留
标准缩写保留英文，如：TLP、DLLP、BAR、LCRC等

### 3. 多义词处理
根据上下文选择最恰当的翻译：
- Lane: "通道"（物理层）或"链路"（逻辑层）
- Credit: "信用量"或"流控信用"

### 4. 专有名词
PCI Express相关专有名词首字母大写：
- Root Complex
- Transaction Layer
- Configuration Space

---

## 参考文档

1. PCIe Skill手册: `/home/ai/.openclaw/skills/pcie/SKILL.md`
2. PCI Express Base Specification 3.0
3. MindShare PCI Express Technology 3.0

### Linux内核术语

| 英文术语 | 中文翻译 | 说明 |
|----------|----------|------|
| struct pci_dev | PCI设备结构体 | Linux内核PCI设备表示 |
| struct pci_bus | PCI总线结构体 | Linux内核PCI总线表示 |
| struct pci_driver | PCI驱动结构体 | Linux内核PCI驱动 |
| struct pcie_link_state | PCIe链路状态结构体 | ASPM链路状态 |
| struct msi_desc | MSI描述符结构体 | MSI中断描述 |
| pci_read_config_byte | 读配置字节 | 读取配置空间1字节 |
| pci_read_config_word | 读配置字 | 读取配置空间2字节 |
| pci_read_config_dword | 读配置双字 | 读取配置空间4字节 |
| pci_write_config_byte | 写配置字节 | 写入配置空间1字节 |
| pci_write_config_word | 写配置字 | 写入配置空间2字节 |
| pci_write_config_dword | 写配置双字 | 写入配置空间4字节 |
| pci_enable_device | 启用PCI设备 | 设备初始化 |
| pci_disable_device | 禁用PCI设备 | 设备关闭 |
| pci_set_master | 设置总线主控 | 启用DMA功能 |
| pci_enable_msi | 启用MSI中断 | MSI初始化 |
| pci_enable_msix_range | 启用MSI-X中断范围 | MSI-X初始化 |
| pci_set_power_state | 设置电源状态 | 电源管理 |
| pci_save_state | 保存PCI状态 | 电源管理保存 |
| pci_restore_state | 恢复PCI状态 | 电源管理恢复 |
| pcie_set_readrq | 设置读请求大小 | 性能优化 |
| pcie_get_minimum_link | 获取最小链路 | 链路信息查询 |
| pcie_bandwidth_available | 可用PCIe带宽 | 带宽查询 |
| pci_alloc_irq_vectors | 分配IRQ向量 | 中断向量分配 |
| pci_free_irq_vectors | 释放IRQ向量 | 中断向量释放 |
| pci_irq_vector | 获取PCI IRQ向量 | 中断向量获取 |
| pci_request_irq | 请求PCI IRQ | 中断请求 |
| pci_free_irq | 释放PCI IRQ | 中断释放 |

---

*术语表版本: v1.1*  
*更新日期: 2026-03-17*  
*更新内容: 添加Linux内核术语章节*
