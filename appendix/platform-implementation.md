# 平台特定实现补充：飞腾PCIe与RK3588 PHY

## 概述

本文档结合《PCI Express Technology 3.0》原文，深度分析飞腾(Phytium)平台PCIe Host控制器驱动和RK3588 PCIe PHY的实现，帮助读者理解PCIe规范在实际硬件平台上的应用。

---

## 第一部分：飞腾PCIe EP控制器实现

### 1.1 硬件架构概述

飞腾PCIe控制器支持Endpoint(EP)模式，主要用于飞腾处理器作为PCIe设备连接到其他主机的场景。

**与书中第2章PCIe架构的对应**:

书中第2章介绍的PCIe架构组件：
```
Root Complex ←→ Switch ←→ Endpoint
                    ↓
                 Endpoint (飞腾)
```

飞腾PCIe EP在系统中作为Endpoint存在，通过PCIe链路与上游的Root Complex或Switch连接。

### 1.2 配置空间实现（对应书中第3章）

#### 寄存器映射

飞腾PCIe EP的配置空间寄存器映射与书中第3章介绍的PCIe配置空间对应：

```c
// 来自 pcie-phytium-register.h
#define PHYTIUM_PCI_VENDOR_ID			0x98
#define PHYTIUM_PCI_DEVICE_ID			0x9a
#define PHYTIUM_PCI_REVISION_ID			0x9c
#define PHYTIUM_PCI_CLASS_PROG			0x9d
#define PHYTIUM_PCI_CLASS_DEVICE		0x9e
```

**与书中表3-1配置空间头部格式的对应**:

| 书中字段 | 飞腾寄存器 | 偏移地址 |
|----------|-----------|----------|
| Vendor ID | PHYTIUM_PCI_VENDOR_ID | 0x98 |
| Device ID | PHYTIUM_PCI_DEVICE_ID | 0x9a |
| Revision ID | PHYTIUM_PCI_REVISION_ID | 0x9c |
| Class Code | PHYTIUM_PCI_CLASS_DEVICE | 0x9e |

**代码示例** - 配置空间写入（对应书中3.5节配置请求）：

```c
// 来自 pcie-phytium-ep.c
static int phytium_pcie_ep_write_header(struct pci_epc *epc, u8 fn,
					struct pci_epf_header *hdr)
{
	struct phytium_pcie_ep *priv = epc_get_drvdata(epc);
	
	// 写入Vendor ID和Device ID
	phytium_pcie_writew(priv, fn, PHYTIUM_PCI_VENDOR_ID, hdr->vendorid);
	phytium_pcie_writew(priv, fn, PHYTIUM_PCI_DEVICE_ID, hdr->deviceid);
	
	// 写入Revision ID
	phytium_pcie_writeb(priv, fn, PHYTIUM_PCI_REVISION_ID, hdr->revid);
	
	// 写入Class Code
	phytium_pcie_writew(priv, fn, PHYTIUM_PCI_CLASS_DEVICE,
			    hdr->subclass_code | (hdr->baseclass_code << 8));
	// ...
}
```

**与书中内容的对应**:
- 书中第3章介绍的配置空间访问机制，在飞腾实现中通过`phytium_pcie_writeb/writew/writel`函数完成
- 这些函数内部实现了Type 0配置请求（书中3.5.1节）

### 1.3 BAR实现（对应书中第4章）

#### BAR寄存器映射

```c
// 来自 pcie-phytium-register.h
#define PHYTIUM_PCI_BAR_0			0xe4
#define PHYTIUM_PCI_BAR(bar_num)		(0xe4 + bar_num * 4)
#define	 BAR_IO_TYPE				(1 << 0)
#define	 BAR_MEM_TYPE				(0 << 0)
#define	 BAR_MEM_64BIT				(1 << 2)
#define	 BAR_MEM_PREFETCHABLE			(1 << 3)
```

**与书中第4章BAR的对应**:

书中第4章介绍的BAR格式（图4-2）：
```
Bit 0: 0=Memory, 1=IO
Bit 1: Reserved (0)
Bit 2: 0=32-bit, 1=64-bit
Bit 3: 0=Non-prefetchable, 1=Prefetchable
Bits 4-31: Base Address
```

飞腾实现中的宏定义与书中格式完全对应：
- `BAR_IO_TYPE` (bit 0) - 对应书中IO Space指示
- `BAR_MEM_64BIT` (bit 2) - 对应书中64-bit指示
- `BAR_MEM_PREFETCHABLE` (bit 3) - 对应书中Prefetchable指示

**代码示例** - BAR设置（对应书中4.2节）：

```c
// 来自 pcie-phytium-ep.c
static int phytium_pcie_ep_set_bar(struct pci_epc *epc, u8 fn,
				   struct pci_epf_bar *epf_bar)
{
	// ...
	if ((flags & PCI_BASE_ADDRESS_SPACE) == PCI_BASE_ADDRESS_SPACE_IO) {
		setting = BAR_IO_TYPE;  // IO空间
		trsl_param = TRSL_ID_IO;
	} else {
		setting = BAR_MEM_TYPE;  // 存储器空间
		
		if (flags & PCI_BASE_ADDRESS_MEM_TYPE_64)
			setting |= BAR_MEM_64BIT;  // 64-bit BAR
		
		if (flags & PCI_BASE_ADDRESS_MEM_PREFETCH)
			setting |= BAR_MEM_PREFETCHABLE;  // 可预取
		
		trsl_param = TRSL_ID_MASTER;
	}
	
	// 写入BAR寄存器
	phytium_pcie_writel(priv, fn, PHYTIUM_PCI_BAR(barno), setting);
	// ...
}
```

**关键对应点**:
- 书中4.2节介绍的BAR类型判断（IO/Memory、32/64-bit、Prefetchable）在代码中通过标志位检查实现
- 飞腾实现支持书中介绍的所有BAR类型

### 1.4 地址转换（对应书中第4章）

#### 窗口机制

飞腾PCIe EP实现了地址转换窗口，用于将PCIe地址空间映射到本地AXI地址空间。

```c
// 来自 pcie-phytium-register.h
#define PHYTIUM_PCI_WIN0_BASE			0x600
#define PHYTIUM_PCI_WIN0_SRC_ADDR0(table)	(PHYTIUM_PCI_WIN0_BASE + 0X20 * table + 0x0)
#define PHYTIUM_PCI_WIN0_TRSL_ADDR0(table)	(PHYTIUM_PCI_WIN0_BASE + 0X20 * table + 0x8)
#define PHYTIUM_PCI_WIN0_TRSL_PARAM(table)	(PHYTIUM_PCI_WIN0_BASE + 0X20 * table + 0x10)
```

**与书中第4章地址路由的对应**:

书中4.6节介绍的地址路由机制：
```
PCIe地址 → 桥接器地址窗口检查 → 匹配 → 转发到下游
```

飞腾实现中的地址转换：
```c
// 来自 pcie-phytium-ep.c
static int phytium_pcie_ep_set_bar(struct pci_epc *epc, u8 fn,
				   struct pci_epf_bar *epf_bar)
{
	// ...
	// 计算ATR大小
	atr_size = fls64(sz - 1) - 1;
	
	// 配置源地址（PCIe地址）
	src_addr0 = ATR_IMPL | ((atr_size & ATR_SIZE_MASK) << ATR_SIZE_SHIFT);
	
	// 配置目标地址（本地物理地址）
	trsl_addr0 = (lower_32_bits(epf_bar->phys_addr) & TRSL_ADDR_32_12_MASK);
	trsl_addr1 = upper_32_bits(epf_bar->phys_addr);
	
	// 写入转换参数
	phytium_pcie_writel(priv, fn, PHYTIUM_PCI_WIN0_SRC_ADDR0(barno), src_addr0);
	phytium_pcie_writel(priv, fn, PHYTIUM_PCI_WIN0_TRSL_ADDR0(barno), trsl_addr0);
	phytium_pcie_writel(priv, fn, PHYTIUM_PCI_WIN0_TRSL_PARAM(barno), trsl_param);
}
```

**关键对应点**:
- 书中4.4节介绍的Base/Limit寄存器在飞腾实现中对应SRC_ADDR和TRSL_ADDR寄存器
- 地址转换实现了书中4.6节介绍的地址路由功能

### 1.5 MSI中断实现（对应书中第17章）

#### MSI能力寄存器

```c
// 来自 pcie-phytium-register.h
#define PHYTIUM_PCI_INTERRUPT_PIN		0xa8
#define	 MSI_DISABLE				(1 << 3)
#define	 MSI_NUM_MASK				(0x7)
#define	 MSI_NUM_SHIFT				4
#define	 MSI_MASK_SUPPORT			(1 << 7)
#define PHYTIUM_PCI_CF_MSI_BASE			0x10e0
#define PHYTIUM_PCI_CF_MSI_CONTROL		0x10e2
```

**与书中第17章MSI的对应**:

书中17.5节介绍的MSI能力结构：
```
MSI Capability ID
Message Control (MMC, MME, Enable)
Message Address (32/64-bit)
Message Data
```

飞腾实现中的MSI配置：

```c
// 来自 pcie-phytium-ep.c
static int phytium_pcie_ep_set_msi(struct pci_epc *epc, u8 fn, u8 mmc)
{
	struct phytium_pcie_ep *priv = epc_get_drvdata(epc);
	u16 flags = 0;
	
	// 设置MSI向量数量 (对应书中Message Control的MMC字段)
	flags = (mmc & MSI_NUM_MASK) << MSI_NUM_SHIFT;
	
	// 禁用MSI掩码支持
	flags &= ~MSI_MASK_SUPPORT;
	
	phytium_pcie_writew(priv, fn, PHYTIUM_PCI_INTERRUPT_PIN, flags);
	return 0;
}
```

**MSI发送实现**（对应书中17.5节MSI发送）：

```c
static int phytium_pcie_ep_send_msi_irq(struct phytium_pcie_ep *priv, u8 fn,
						  u8 interrupt_num)
{
	// 读取MSI配置
	flags = phytium_pcie_readw(priv, fn, cap + PCI_MSI_FLAGS);
	
	// 检查MSI是否启用
	if (!(flags & PCI_MSI_FLAGS_ENABLE))
		return -EINVAL;
	
	// 获取MSI向量数量 (对应书中MME字段)
	mme = (flags & PCI_MSI_FLAGS_QSIZE) >> 4;
	msi_count = 1 << mme;
	
	// 构造MSI数据 (对应书中Message Data)
	data_mask = msi_count - 1;
	data = phytium_pcie_readw(priv, fn, cap + PCI_MSI_DATA_64);
	data = (data & ~data_mask) | ((interrupt_num - 1) & data_mask);
	
	// 获取MSI地址 (对应书中Message Address)
	pci_addr = phytium_pcie_readl(priv, fn, cap + PCI_MSI_ADDRESS_HI);
	pci_addr <<= 32;
	pci_addr |= phytium_pcie_readl(priv, fn, cap + PCI_MSI_ADDRESS_LO);
	
	// 配置地址转换并发送MSI
	// ...
}
```

**关键对应点**:
- 书中17.5.1节介绍的MSI能力结构在飞腾实现中通过PHYTIUM_PCI_CF_MSI_BASE等寄存器实现
- MSI发送过程（书中17.5.3节）在代码中通过构造Memory Write TLP实现

### 1.6 Device Tree配置

```dts
// 来自 phytium,phytium-pcie-ep.txt
ep0: ep@0x29030000 {
	compatible = "phytium,pcie-ep-1.0";
	reg = <0x0 0x29030000 0x0 0x10000>,    // 控制器寄存器
	      <0x11 0x00000000 0x1 0x00000000>, // AXI接口区域
	      <0x0 0x29101000 0x0 0x1000>;     // HPB寄存器
	reg-names = "reg", "mem", "hpb";
	max-outbound-regions = <3>;  // 最大outbound区域数
	max-functions = /bits/ 8 <1>; // 最大功能数
};
```

**与书中内容的对应**:
- `max-outbound-regions` 对应书中第4章介绍的地址转换窗口数量
- `max-functions` 对应书中第3章介绍的Function概念

---

## 第二部分：RK3588 PCIe PHY实现

### 2.1 PHY概述（对应书中第11-13章）

RK3588 PCIe PHY负责PCIe物理层的信号处理，包括：
- 串行化/解串行化（书中11.2节）
- 时钟生成和恢复（书中11.3节）
- 链路训练支持（书中第14章）

### 2.2 PHY寄存器配置

```c
// 来自 phy-rockchip-pcie.c
#define PHY_CFG_DATA_SHIFT    7
#define PHY_CFG_ADDR_SHIFT    1
#define PHY_CFG_DATA_MASK     0xf
#define PHY_CFG_ADDR_MASK     0x3f
#define PHY_CFG_PLL_LOCK      0x10
#define PHY_CFG_CLK_TEST      0x10
#define PHY_CFG_CLK_SCC       0x12
#define PHY_CFG_SEPE_RATE     BIT(3)
#define PHY_CFG_PLL_100M      BIT(3)
#define PHY_PLL_LOCKED        BIT(9)
#define PHY_PLL_OUTPUT        BIT(10)
```

**与书中第13章电气特性的对应**:

书中13.6节介绍的发送器规范：
- 时钟频率
- 信号摆幅
- 去加重

RK3588 PHY通过寄存器配置这些参数：

```c
// PHY配置写入函数
static inline void phy_wr_cfg(struct rockchip_pcie_phy
static inline void phy_wr_cfg(struct rockchip_pcie_phy *rk_phy,
			      u32 addr, u32 data)
{
	// 配置PHY内部寄存器
	regmap_write(rk_phy->reg_base, rk_phy->phy_data->pcie_conf,
		     HIWORD_UPDATE(data,
				   PHY_CFG_DATA_MASK,
				   PHY_CFG_DATA_SHIFT) |
		     HIWORD_UPDATE(addr,
				   PHY_CFG_ADDR_MASK,
				   PHY_CFG_ADDR_SHIFT));
	// ...
}
```

### 2.3 PLL锁定过程（对应书中第14章链路训练）

**与书中14.4节Detect状态的对应**:

书中介绍的链路训练Detect状态用于检测链路对端是否存在。

RK3588 PHY的PLL锁定过程：

```c
// 来自 phy-rockchip-pcie.c
static int rockchip_pcie_phy_power_on(struct phy *phy)
{
	// ...
	// 释放PHY复位（对应书中退出复位状态）
	err = reset_control_deassert(rk_phy->phy_rst);
	
	// 等待PLL锁定（对应书中时钟恢复）
	timeout = jiffies + msecs_to_jiffies(1000);
	err = -EINVAL;
	while (time_before(jiffies, timeout)) {
		regmap_read(rk_phy->reg_base,
			    rk_phy->phy_data->pcie_status,
			    &status);
		if (status & PHY_PLL_LOCKED) {
			dev_dbg(&phy->dev, "pll locked!\n");
			err = 0;
			break;
		}
		msleep(20);
	}
	// ...
	
	// 配置时钟测试（对应书中11.2节发送逻辑）
	phy_wr_cfg(rk_phy, PHY_CFG_CLK_TEST, PHY_CFG_SEPE_RATE);
	phy_wr_cfg(rk_phy, PHY_CFG_CLK_SCC, PHY_CFG_PLL_100M);
	
	// 等待PLL输出使能完成
	while (time_before(jiffies, timeout)) {
		regmap_read(rk_phy->reg_base,
			    rk_phy->phy_data->pcie_status,
			    &status);
		if (!(status & PHY_PLL_OUTPUT)) {
			dev_dbg(&phy->dev, "pll output enable done!\n");
			err = 0;
			break;
		}
		msleep(20);
	}
}
```

**关键对应点**:
- PLL锁定过程对应书中14.4节Detect状态的时钟恢复阶段
- PHY_CFG_SEPE_RATE配置对应书中13.6节的发送器去加重设置
- PHY_CFG_PLL_100M配置对应书中13.5节的时钟要求

### 2.4 通道状态检测（对应书中第14章）

```c
// 来自 phy-rockchip-pcie.c
#define PHY_LANE_A_STATUS     0x30
#define PHY_LANE_B_STATUS     0x31
#define PHY_LANE_C_STATUS     0x32
#define PHY_LANE_D_STATUS     0x33
#define PHY_LANE_RX_DET_SHIFT 11
#define PHY_LANE_RX_DET_TH    0x1
```

**与书中14.4节Polling状态的对应**:

书中Polling状态用于位锁定和极性检测。

RK3588 PHY的通道状态寄存器：
- PHY_LANE_A/B/C/D_STATUS - 对应4个通道的状态
- PHY_LANE_RX_DET - 接收检测状态（对应书中Detect状态的接收检测）

### 2.5 电源管理（对应书中第16章）

```c
// PHY电源关闭
static int rockchip_pcie_phy_power_off(struct phy *phy)
{
	// ...
	// 设置通道为空闲状态（对应书中L0s/L1状态进入）
	regmap_write(rk_phy->reg_base,
		     rk_phy->phy_data->pcie_laneoff,
		     HIWORD_UPDATE(PHY_LANE_IDLE_OFF,
				   PHY_LANE_IDLE_MASK,
				   PHY_LANE_IDLE_A_SHIFT + inst->index));
	
	// 断言PHY复位（进入低功耗状态）
	if (--rk_phy->pwr_cnt)
		goto err_out;
	
	err = reset_control_assert(rk_phy->phy_rst);
	// ...
}
```

**与书中16.4节链路电源管理的对应**:

- PHY_LANE_IDLE_OFF 对应书中L0s状态的电气空闲
- reset_control_assert 对应书中L2/L3状态的时钟关闭

### 2.6 Device Tree配置

```dts
// 来自 rockchip-pcie-phy.txt
pcie_phy: pcie-phy {
	compatible = "rockchip,rk3399-pcie-phy";
	#phy-cells = <0>;
	clocks = <&cru SCLK_PCIEPHY_REF>;  // PHY参考时钟
	clock-names = "refclk";
	resets = <&cru SRST_PCIEPHY>;      // PHY复位
	reset-names = "phy";
};
```

**与书中内容的对应**:
- `clocks` 对应书中13.5节的参考时钟要求
- `resets` 对应书中18章的复位机制

---

## 第三部分：对比总结

### 3.1 架构对比

| 特性 | 飞腾PCIe EP | RK3588 PCIe PHY |
|------|-------------|-----------------|
| 角色 | Endpoint (设备端) | PHY (物理层) |
| 主要功能 | 配置空间、BAR、MSI | 信号处理、时钟、训练 |
| 对应书中章节 | Ch2-5, 17 | Ch11-14 |
| 软件接口 | pci_epc/pci_epf | phy framework |

### 3.2 关键实现对比

#### 配置空间访问
- **飞腾**: 通过内部寄存器直接访问（EP模式）
- **RK3588**: 通过Host控制器间接访问（PHY层不直接处理）

#### 地址转换
- **飞腾**: 实现ATR（Address Translation）机制
- **RK3588**: PHY层不涉及地址转换

#### 中断处理
- **飞腾**: 实现MSI/MSI-X发送（EP端）
- **RK3588**: PHY层传递信号，不处理中断逻辑

#### 电源管理
- **飞腾**: 支持D0-D3设备状态
- **RK3588**: 支持L0s/L1链路状态

### 3.3 与书中内容的映射

```
《PCI Express Technology 3.0》
        ↓
   各章节理论
        ↓
   ┌─────────┬─────────┐
   ↓         ↓         ↓
飞腾PCIe   RK3588    Linux内核
  EP       PHY       框架
   ↓         ↓         ↓
配置空间   物理层    驱动接口
BAR/MSI   时钟/训练  设备管理
```

---

## 第四部分：实际应用建议

### 4.1 驱动开发参考

1. **配置空间操作**: 参考飞腾实现理解EP模式下的配置空间处理
2. **PHY初始化**: 参考RK3588实现理解物理层初始化流程
3. **地址映射**: 结合书中第4章和飞腾BAR实现理解地址转换

### 4.2 调试技巧

1. **链路训练问题**: 检查RK3588 PHY的PLL锁定状态和通道检测
2. **枚举问题**: 检查飞腾EP的配置空间响应
3. **性能问题**: 结合书中第7章QoS和飞腾ATR配置优化

### 4.3 学习路径

1. 先阅读书中第2-5章理解PCIe基础架构
2. 结合飞腾代码理解EP模式实现
3. 阅读书中第11-14章理解物理层
4. 结合RK3588代码理解PHY实现
5. 阅读书中第15-19章理解高级功能
6. 结合Linux内核代码理解软件框架

---

## 参考代码路径

### 飞腾PCIe EP
- `/home/ai/dev/04-sdk/phytium-linux-kernel/drivers/pci/controller/pcie-phytium-ep.c`
- `/home/ai/dev/04-sdk/phytium-linux-kernel/drivers/pci/controller/pcie-phytium-register.h`
- `/home/ai/dev/04-sdk/phytium-linux-kernel/Documentation/devicetree/bindings/pci/phytium,phytium-pcie-ep.txt`

### RK3588 PCIe PHY
- `/home/ai/dev/04-sdk/phytium-linux-kernel/drivers/phy/rockchip/phy-rockchip-pcie.c`
- `/home/ai/dev/04-sdk/phytium-linux-kernel/Documentation/devicetree/bindings/phy/rockchip-pcie-phy.txt`

---

*文档创建日期: 2026-03-17*  
*对应书籍: MindShare PCI Express Technology 3.0*  
*参考内核: Phytium Linux Kernel*

---

## 附录图片

本节提供PCIe技术相关的参考图表和实现示意图。

### 图A-1 PCIe信号定义
![图A-1 PCIe信号定义](../images/图A-1.PCIe信号定义.png)
*PCIe信号定义图，展示各信号线的功能和定义*

### 图A-2 引脚分配
![图A-2 引脚分配](../images/图A-2.引脚分配.png)
*PCIe连接器引脚分配图，详细标注各引脚功能*

### 图A-3 连接器尺寸
![图A-3 连接器尺寸](../images/图A-3.连接器尺寸.png)
*PCIe连接器物理尺寸和机械规格*

### 图A-4 时序参数
![图A-4 时序参数](../images/图A-4.时序参数.png)
*PCIe信号时序参数表，包含关键时序要求*

### 图A-5 电气规范
![图A-5 电气规范](../images/图A-5.电气规范.png)
*PCIe电气特性规范，包括电压、电流和阻抗要求*

### 图A-6 测试点
![图A-6 测试点](../images/图A-6.测试点.png)
*PCIe信号测试点位置和测试方法说明*

### 图A-7 飞腾PCIe
![图A-7 飞腾PCIe](../images/图A-7.飞腾PCIe.jpg)
*飞腾PCIe EP控制器架构示意图*

### 图A-8 RK3588 PHY
![图A-8 RK3588 PHY](../images/图A-8.RK3588_PHY.jpg)
*RK3588 PCIe PHY物理层实现框图*

### 图A-9 Linux驱动
![图A-9 Linux驱动](../images/图A-9.Linux驱动.jpg)
*Linux PCIe驱动架构和软件栈示意图*

### 图A-10 调试技巧
![图A-10 调试技巧](../images/图A-10.调试技巧.jpg)
*PCIe调试技巧和常见问题排查方法*
