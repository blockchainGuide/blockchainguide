# 汇总节点规范

[Rollup 节点](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#rollup-node)是负责从 L1 区块（及其关联的[收据）](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#receipt)[派生 L2 链的](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#L2-chain-derivation)组件。

Rollup 节点中派生 L2 链的部分称为[rollup 驱动程序](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#rollup-driver)。本文档目前仅涉及 rollup 驱动程序的规范。

**目录**

- 司机
  - [推导](https://github.com/ethereum-optimism/optimism/blob/develop/specs/rollup-node.md#derivation)
- L2输出RPC方法
  - [输出方法API](https://github.com/ethereum-optimism/optimism/blob/develop/specs/rollup-node.md#output-method-api)

## 司机

[Rollup 节点](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#rollup-node)中的[driver](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#rollup-driver)的任务 是管理[派生](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#L2-chain-derivation)过程：

- 跟踪 L1 头块
- 跟踪L2链同步进度
- 当新输入可用时迭代推导步骤

### 推导

此过程分三个步骤进行：

1. 从最后一个 L2 区块顶部的 L1 链中选择输入：区块列表，包含交易以及相关数据和收据。
2. 读取 L1 信息、存款和排序批次，以生成有效[负载属性](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#payload-attributes) （本质上[是没有输出属性的块](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#block)）。
3. 将有效负载属性传递给[执行引擎](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#execution-engine)，以便可以计算L2块（包括[输出块属性）。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#block)

虽然这个过程在概念上是从 L1 链到 L2 链的纯函数，但实际上是增量的。每当新的 L1 块添加到 L1 链时，L2 链就会扩展。类似地，每当 L1 链重新组织时，L2 链也会[重新组织](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#re-organization)。

有关 L2 块推导的完整规范，请参阅[L2 块推导文档](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md)。

## L2输出RPC方法

Rollup 节点有自己的 RPC 方法，`optimism_outputAtBlock`该方法返回与[L2 输出 root](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md#l2-output-commitment-construction)对应的 32 字节哈希。

### 输出方法API

这里的输入和返回类型是由[引擎 API 规范](https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md#structures)定义的）。

- 方法：`optimism_outputAtBlock`
- 参数：
  1. `blockNumber`: `QUANTITY`, 64 位 - L2 整数块号
     OR - 、、 或`String`之一。`"safe"``"latest"``"pending"`
- 返回：
  1. `version`: `DATA`, 32 字节 - 输出根版本号，从 0 开始。
  2. `l2OutputRoot`: `DATA`, 32 字节 - 输出根。