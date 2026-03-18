# 第11章 物理层 - 逻辑（Gen1和Gen2）

## 上一章回顾

上一章描述了 Ack/Nak 协议：一种自动的、基于硬件的机制，用于确保 TLP（事务层包）在链路上的可靠传输。Ack DLLP（数据链路层包）确认 TLP 的正确接收，而 Nak DLLP 则表示传输错误。该章节描述了正常操作规则以及与 TLP 和 DLLP 错误相关的错误恢复机制。

## 本章内容

本章描述物理层的逻辑子模块。该模块为串行传输准备数据包并恢复数据。完成这一任务需要多个步骤，本章将详细描述这些步骤。本章涵盖与 Gen1 和 Gen2 协议相关的逻辑，这些协议使用 8b/10b 编码。Gen3 的逻辑不使用 8b/10b 编码，将在名为"物理层 - 逻辑（Gen3）"的章节中单独描述。

## 下一章预告

下一章将描述第三代 PCIe（Gen3）的物理层特性。主要变化包括无需将频率加倍即可将带宽相对于 Gen2 加倍的能力，这是通过消除对 8b/10b 编码的需求实现的。在 Gen3 速度下，需要更稳健的信号补偿。实现这些变化比预期的要复杂得多。

---

## 物理层概述

本物理层概述介绍了 Gen1、Gen2 和 Gen3 实现之间的关系。此后，重点将放在与 Gen1 和 Gen2 相关的逻辑物理层实现上。Gen3 的逻辑物理层实现将在下一章中描述。

物理层位于外部物理链路与数据链路层之间接口的底部。它将来自数据链路层的出站数据包转换为串行比特流，并时钟同步到链路的所有通道（Lane）上。该层还从链路的所有通道恢复比特流。接收逻辑将比特反序列化为符号流，重新组装数据包，并将 TLP 和 DLLP 转发给数据链路层。

各层的内容是概念性的，并不定义精确的逻辑块，但设计师如果将设计按层划分以匹配规范，他们的实现将受益，因为不断增长的速率对物理层的影响大于其他层。按层职责划分设计允许物理层适应更高的时钟速率，同时尽可能少地改变其他层。

PCIe 规范 3.0 版不使用特定术语来区分由规范版本定义的不同传输速率。考虑到这一点，本书定义并使用以下术语：

- **Gen1** - 第一代 PCIe（修订版 1.x），工作速率为 2.5 GT/s
- **Gen2** - 第二代（修订版 2.x），工作速率为 5.0 GT/s
- **Gen3** - 第三代（修订版 3.x），工作速率为 8.0 GT/s

物理层由两个子模块组成：逻辑部分和电气部分。两者都包含独立的发送和接收逻辑，允许双工通信。

---

## 观察说明

规范描述了物理层的功能，但在实现细节上有意保持模糊。显然，规范编写者不愿意提供细节或示例实现，因为他们希望为各个供应商留出空间，通过巧妙或有创意的逻辑版本来增加价值。对于我们的讨论，示例是不可或缺的，因此选择了一个来说明概念。需要明确的是，这个示例未经测试或验证，设计师也不应感到有必要以这种方式实现物理层。

---

## Linux内核实现参考

### 物理层寄存器访问

在Linux内核中，PCIe物理层寄存器通过内存映射I/O访问。以Phytium PCIe控制器为例：

```c
// Phytium PCIe EP寄存器访问
#define PHYTIUM_PCIE_FUNC_BASE(fn)  (((fn) << 14) & GENMASK(16, 14))

static inline void phytium_pcie_writel(struct phytium_pcie_ep *priv, 
                                        u8 fn, u32 reg, u32 value)
{
    writel(value, priv->reg_base + PHYTIUM_PCIE_FUNC_BASE(fn) + reg);
}
c
// DesignWare PCIe链路宽度配置
#define PCIE_PORT_LINK_CONTROL    0x710
#define PORT_LINK_MODE_MASK       GENMASK(21, 16)
#define PORT_LINK_MODE_1_LANES    PORT_LINK_MODE(0x1)
#define PORT_LINK_MODE_4_LANES    PORT_LINK_MODE(0x7)
#define PORT_LINK_MODE_8_LANES    PORT_LINK_MODE(0xf)

// 配置为x4链路
dw_pcie_writel_dbi(pci, PCIE_PORT_LINK_CONTROL, PORT_LINK_MODE_4_LANES);
```

### 加扰器控制

虽然加扰器通常由硬件自动管理，但软件可以通过链路控制寄存器禁用加扰（仅用于测试）：

```c
// 通过TS1/TS2中的训练控制字段禁用加扰
// 设置Training Control字段的Bit 3 (Disable Scrambling)

来自多路复用器块的数据包字节流
┌─────────────────────────────────────┐
│ Character 7                         │
│ Character 6                         │
│ Character 5                         │
│ Character 4                         │
│ Character 3                         │
│ Character 2                         │
│ Character 1                         │
│ Character 0                         │
└─────────────────────────────────────┘
              ↓
         x1 字节分段
              ↓
┌─────────────────────────────────────┐
│ Character 2  →  到加扰器           │
│ Character 1  →  到加扰器           │
│ Character 0  →  到加扰器           │
└─────────────────────────────────────┘

来自多路复用器块的数据包双字流

Character 12  Character 13  Character 14  Character 15
     ↓             ↓             ↓             ↓
  通道 0        通道 1        通道 2        通道 3

Character 8   Character 9   Character 10  Character 11
     ↓             ↓             ↓             ↓
  通道 0        通道 1        通道 2        通道 3

Character 4   Character 5   Character 6   Character 7
     ↓             ↓             ↓             ↓
  通道 0        通道 1        通道 2        通道 3

Character 0   Character 1   Character 2   Character 3
     ↓             ↓             ↓             ↓
  通道 0        通道 1        通道 2        通道 3

时间 →
通道 0: STP - TLP - END - SDP - DLLP - END - STP - TLP - END - COM - SKP - STP - TLP - END - Idle - Idle - Idle

通道:    0      1      2      3
        ┌──────┬──────┬──────┬──────┐
时间 →  │ STP  │ TLP  │ TLP  │ TLP  │
        │ LCRC │ LCRC │ LCRC │ END  │
        │ COM  │ SKP  │ SKP  │ SKP  │  ← SKIP 有序集
        │ SDP  │ DLLP │ DLLP │ END  │
        │ Idle │ Idle │ Idle │ Idle │  ← 逻辑空闲
        └──────┴──────┴──────┴──────┘

通道:   0      1      2      3      4      5      6      7
       ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
       │ STP  │ TLP  │ TLP  │ TLP  │ TLP  │ TLP  │ TLP  │ TLP  │
       │ LCRC │ LCRC │ LCRC │ LCRC │ LCRC │ LCRC │ LCRC │ END  │
       │ COM  │ SKP  │ SKP  │ SKP  │ SKP  │ SKP  │ SKP  │ SKP  │ ← SKIP 有序集
       │ SDP  │ DLLP │ DLLP │ DLLP │ DLLP │ DLLP │ END  │      │
       │ STP  │ TLP  │ TLP  │ END  │ PAD  │ PAD  │ PAD  │ PAD  │ ← PAD 填充
       │ Idle │ Idle │ Idle │ Idle │ Idle │ Idle │ Idle │ Idle │ ← 逻辑空闲
       └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘

G(x) = X¹⁶ + X⁵ + X⁴ + X³ + 1
```

```
加扰器结构 (16-bit LFSR)

X0 ──→ X1 ──→ X2 ──→ ... ──→ X15 ──→ XOR ──→ 反馈
                                     ↑
                                     └── XOR ←── X3
                                         ↑
                                         └── XOR ←── X4
                                             ↑
                                             └── XOR ←── X5

LFSR 输出: 8 位并行
数据输入: 8 位字符
输出: Scrambled Data = Data XOR LFSR_Output

工作时钟: 8 倍字节时钟频率 (Gen1: 250MHz 字节时钟 → 2GHz LFSR 时钟)

8b/10b 命名法示例

8b 字符: 0110 1010 (6Ah)

步骤 1: 分成子块
  HGF = 011 (3位)
  EDCBA = 01010 (5位)

步骤 2: 调换子块位置
  EDCBA HGF = 01010 011

步骤 3: 转换为十进制
  01010 = 10
  011 = 3

步骤 4: 最终表示
  D10.3 (数据字符)

n = 1538 + (最大数据包有效载荷大小 + 28)
c
// 来自 drivers/phy/rockchip/phy-rockchip-pcie.c
#define PHY_CFG_DATA_SHIFT    7
#define PHY_CFG_ADDR_SHIFT    1
#define PHY_CFG_DATA_MASK     0xf
#define PHY_CFG_ADDR_MASK     0x3f
#define PHY_CFG_PLL_LOCK      0x10    // PLL锁定配置
#define PHY_CFG_CLK_TEST      0x10    // 时钟测试
#define PHY_CFG_CLK_SCC       0x12    // 时钟配置
#define PHY_CFG_SEPE_RATE     BIT(3)  // 信号速率
#define PHY_CFG_PLL_100M      BIT(3)  // 100M PLL
#define PHY_PLL_LOCKED        BIT(9)  // PLL锁定状态
#define PHY_PLL_OUTPUT        BIT(10) // PLL输出状态
c
// PHY配置写入（对应书中物理层寄存器访问）
static inline void phy_wr_cfg(struct rockchip_pcie_phy *rk_phy,
                              u32 addr, u32 data)
{
    regmap_write(rk_phy->reg_base, rk_phy->phy_data->pcie_conf,
                 HIWORD_UPDATE(data,
                               PHY_CFG_DATA_MASK,
                               PHY_CFG_DATA_SHIFT) |
                 HIWORD_UPDATE(addr,
                               PHY_CFG_ADDR_MASK,
                               PHY_CFG_ADDR_SHIFT));
    udelay(1);
    // 写使能序列
    regmap_write(rk_phy->reg_base, rk_phy->phy_data->pcie_conf,
                 HIWORD_UPDATE(PHY_CFG_WR_ENABLE,
                               PHY_CFG_WR_MASK,
                               PHY_CFG_WR_SHIFT));
    udelay(1);
    regmap_write(rk_phy->reg_base, rk_phy->phy_data->pcie_conf,
                 HIWORD_UPDATE(PHY_CFG_WR_DISABLE,
                               PHY_CFG_WR_MASK,
                               PHY_CFG_WR_SHIFT));
}

// PHY配置读取
static inline u32 phy_rd_cfg(struct rockchip_pcie_phy *rk_phy,
                             u32 addr)
{
    u32 val;
    regmap_write(rk_phy->reg_base, rk_phy->phy_data->pcie_conf,
                 HIWORD_UPDATE(addr,
                               PHY_CFG_RD_MASK,
                               PHY_CFG_ADDR_SHIFT));
    regmap_read(rk_phy->reg_base,
                rk_phy->phy_data->pcie_status,
                &val);
    return val;
}
c
// 来自 phy-rockchip-pcie.c
static int rockchip_pcie_phy_power_on(struct phy *phy)
{
    // ...
    // 释放PHY复位（对应书中退出复位状态）
    err = reset_control_deassert(rk_phy->phy_rst);
    
    // 等待PLL锁定（对应书中时钟恢复）
    timeout = jiffies + msecs_to_jiffies(1000);
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
    
    // 配置时钟测试（对应书中11.2节发送逻辑）
    phy_wr_cfg(rk_phy, PHY_CFG_CLK_TEST, PHY_CFG_SEPE_RATE);
    phy_wr_cfg(rk_phy, PHY_CFG_CLK_SCC, PHY_CFG_PLL_100M);
    
    // 等待PLL输出使能
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

**与书中内容的对应**：

1. **PLL锁定**（对应14.4节Detect状态）：
   - 等待`PHY_PLL_LOCKED`对应书中时钟恢复完成
   - 超时处理对应书中训练超时机制

2. **时钟配置**（对应13.5节）：
   - `PHY_CFG_PLL_100M`对应书中100MHz参考时钟要求
   - `PHY_CFG_SEPE_RATE`对应书中发送速率配置

### 11.16.3 通道状态检测

```c
// 通道状态寄存器（对应书中14.4节Polling状态）
#define PHY_LANE_A_STATUS     0x30
#define PHY_LANE_B_STATUS     0x31
#define PHY_LANE_C_STATUS     0x32
#define PHY_LANE_D_STATUS     0x33
#define PHY_LANE_RX_DET_SHIFT 11
#define PHY_LANE_RX_DET_TH    0x1
```

**与书中14.4节Polling状态的对应**：
- `PHY_LANE_A/B/C/D_STATUS` 对应4个通道的状态检测
- `PHY_LANE_RX_DET` 对应书中接收检测（Receiver Detection）

### 11.16.4 电源管理

```c
// PHY电源关闭（对应书中16.4节链路电源管理）
static int rockchip_pcie_phy_power_off(struct phy *phy)
{
    // ...
    // 设置通道为空闲状态（对应书中L0s状态）
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

**与书中16.4节的对应**：
- `PHY_LANE_IDLE_OFF` 对应书中L0s状态的电气空闲
- `reset_control_assert` 对应书中L2/L3状态的时钟关闭

### 11.16.5 Device Tree配置

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

**与书中内容的对应**：
- `clocks` 对应书中13.5节的参考时钟要求
- `resets` 对应书中18章的复位机制

### 11.16.6 实际应用建议

**PHY调试技巧**：
1. 检查PLL锁定状态（`PHY_PLL_LOCKED`）
2. 验证通道检测状态（`PHY_LANE_RX_DET`）
3. 监控电源状态转换

**性能优化**：
- 合理配置去加重（`PHY_CFG_SEPE_RATE`）
- 优化PLL锁定等待时间
- 配置适当的电源管理策略

---

*翻译来源: MindShare PCI Express Technology 3.0, Chapter 11: Physical Layer - Logical (Gen1 and Gen2)*
*平台补充: RK3588 PCIe PHY实现参考*
*翻译日期: 2026-03-17*

---

## 本章图片附录

以下是本章相关的原文图片：

### 图11-1.8b10b编码.jpg

![图11-1.8b10b编码.jpg](../images/图11-1.8b10b编码.jpg)

### 图11-2.加扰.jpg

![图11-2.加扰.jpg](../images/图11-2.加扰.jpg)

### 图11-3.有序集.jpg

![图11-3.有序集.jpg](../images/图11-3.有序集.jpg)

### 图11-4.训练序列.jpg

![图11-4.训练序列.jpg](../images/图11-4.训练序列.jpg)

### 图11-5.链路训练.jpg

![图11-5.链路训练.jpg](../images/图11-5.链路训练.jpg)

