# 死磕Uniswap V2

> 深入解析Uniswap V2的核心技术与实现原理

## 系列概述

本系列文章深入剖析Uniswap V2的技术架构，从核心设计理念到实现细节，全面解析V2作为第二代AMM协议的技术创新。

## 系列导航

| 序号 | 标题 | 核心内容 | 状态 |
|:----:|------|----------|:----:|
| **01** | **V2概述与核心原理** | **恒定乘积AMM、核心公式** | 📝 |
| **02** | **Factory与Pair合约** | **合约结构、创建流程** | 📝 |
| **03** | **流动性与LP代币** | **mint/burn、份额计算** | 📝 |
| **04** | **交换机制深度解析** | **swap函数、滑点、Flash Swap** | 📝 |
| **05** | **价格预言机** | **TWAP、价格计算** | 📝 |
| **06** | **Router与路由** | **最佳路径、多跳交易** | 📝 |
| **07** | **安全实践与最佳实践** | **漏洞防护、开发建议** | 📝 |

## V2 vs V1 核心差异

| 特性 | Uniswap V1 | Uniswap V2 |
|------|-----------|-----------|
| **交易对** | ETH/ERC20 | ERC20/ERC20 |
| **流动性** | 单一池子 | 多个独立池子 |
| **Flash Swap** | ❌ | ✅ |
| **价格预言机** | ❌ | ✅ TWAP |
| **协议费用** | 0.3% 固定 | 0.3% 可开关 |
| **路由** | 直接交易 | 多跳路径 |

## V2 vs V3/V4 对比

| 特性 | V2 | V3 | V4 |
|------|----|----|-----|
| **流动性分布** | 均匀(0,∞) | 可集中 | 可集中+Hooks |
| **费用** | 固定0.3% | 多档位 | 动态可编程 |
| **LP代币** | ERC20 | ERC721 NFT | ERC1155可选 |
| **架构** | Factory模式 | Factory模式 | Singleton |
| **扩展性** | 固定功能 | 固定功能 | Hooks可编程 |

## 核心创新点

### 1. 恒定乘积AMM

```
x × y = k
```

- `x`: Token0储备量
- `y`: Token1储备量
- `k`: 恒定乘积常数

### 2. ERC20/ERC20交易

V2支持任意两个ERC20代币之间的直接交易，无需通过ETH中转。

### 3. Flash Swap

无需抵押的闪电贷款，支持原子化套利和清算操作。

### 4. TWAP预言机

基于时间加权的平均价格，提供可靠的价格数据源。

## 技术栈

- **Solidity**: ^0.6.0 (V2合约)
- **Factory**: UNISWAP_V2_FACTORY
- **Router**: UNISWAP_V2_ROUTER_02
- **标准**: ERC20

## 学习路径

建议按顺序阅读本系列文章：

1. **入门**: 先阅读「01-V2概述与核心原理」，理解AMM基础
2. **合约**: 深入「02-Factory与Pair合约」，掌握核心合约
3. **流动性**: 学习「03-流动性与LP代币」，了解LP机制
4. **交换**: 深入「04-交换机制深度解析」，理解交易流程
5. **预言机**: 掌握「05-价格预言机」，了解TWAP原理
6. **路由**: 学习「06-Router与路由」，掌握多跳交易
7. **实践**: 阅读「07-安全实践」，确保开发安全

## 参考资料

- [Uniswap V2 Whitepaper](https://uniswap.org/whitepaper.pdf)
- [Uniswap V2 Core Code](https://github.com/Uniswap/v2-core)
- [Uniswap V2 Periphery](https://github.com/Uniswap/v2-periphery)
- [ERC-20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)

## 贡献

欢迎提交Issue和Pull Request来完善本系列文档。

## 许可证

MIT License
