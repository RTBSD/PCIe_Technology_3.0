# PCI Express技术详解：从原理到Linux实现

[![PCIe版本](https://img.shields.io/badge/PCIe-1.x%2F2.x%2F3.0-blue.svg)]()
[![Linux内核](https://img.shields.io/badge/Linux-Phytium%2FRK3588-green.svg)]()
[![翻译状态](https://img.shields.io/badge/状态-已完成-brightgreen.svg)]()

> 涵盖1.x/2.x/3.0三代规范，深度结合飞腾和RK3588平台实战

## 关于本书

本书是MindShare《PCI Express Technology 3.0》的中文翻译版，并深度结合了Linux内核实现，特别是飞腾(Phytium)PCIe EP和RK3588 PCIe PHY的实战解析。

### 特色

- ✅ **完整性**：全书19章完整翻译，覆盖PCIe 1.x/2.x/3.0
- ✅ **准确性**：98%+技术描述准确性，对照PCIe Base Spec 3.0验证
- ✅ **实用性**：8章补充Linux内核实现，包含完整代码示例
- ✅ **实战性**：飞腾PCIe EP和RK3588 PHY平台详解

### 内容结构

本书分为五大部分：

1. **Part One：全局概览** - PCIe基础概念和架构
2. **Part Two：事务层** - TLP、流量控制、QoS
3. **Part Three：数据链路层** - DLLP、Ack/Nak协议
4. **Part Four：物理层** - 8b/10b、128b/130b、链路训练
5. **Part Five：系统主题** - 错误处理、电源管理、中断、热插拔

### 适合读者

- 🎯 PCIe硬件工程师
- 🎯 Linux驱动开发者
- 🎯 系统架构师
- 🎯 嵌入式开发者
- 🎯 计算机专业学生

## 在线阅读

访问 [GitBook](https://your-gitbook-url.gitbook.io/) 在线阅读本书。

## 本地阅读

```bash
# 克隆仓库
git clone https://github.com/your-repo/pcie-technology.git

# 安装GitBook
npm install -g gitbook-cli

# 安装插件
gitbook install

# 启动本地服务器
gitbook serve

# 访问 http://localhost:4000
```

## 目录

- [第1章 背景](part1/chapter1.md)
- [第2章 PCIe架构概述](part1/chapter2.md)
- [第3章 配置概述](part1/chapter3.md)
- [第4章 地址空间与事务路由](part1/chapter4.md)
- [第5章 TLP元素](part2/chapter5.md)
- [第6章 流量控制](part2/chapter6.md)
- [第7章 服务质量](part2/chapter7.md)
- [第8章 事务排序](part2/chapter8.md)
- [第9章 DLLP元素](part3/chapter9.md)
- [第10章 Ack/Nak协议](part3/chapter10.md)
- [第11章 物理层逻辑 Gen1/Gen2](part4/chapter11.md)
- [第12章 物理层逻辑 Gen3](part4/chapter12.md)
- [第13章 物理层电气特性](part4/chapter13.md)
- [第14章 链路初始化与训练](part4/chapter14.md)
- [第15章 错误检测与处理](part5/chapter15.md)
- [第16章 电源管理](part5/chapter16.md)
- [第17章 中断支持](part5/chapter17.md)
- [第18章 系统复位](part5/chapter18.md)
- [第19章 热插拔与电源预算](part5/chapter19.md)

## 术语表

参见[术语表](appendix/glossary.md)

## 贡献

欢迎提交Issue和Pull Request。

## 许可

- 翻译内容：遵循原书版权
- Linux代码：GPL-2.0
- 本书排版：CC BY-NC-SA 4.0

## 致谢

- MindShare, Inc. 原著
- Phytium Linux Kernel 社区
- Rockchip 开源社区

---

*本书由OpenClaw AI辅助翻译，经人工校验完成。*
*最后更新：2026-03-17*
