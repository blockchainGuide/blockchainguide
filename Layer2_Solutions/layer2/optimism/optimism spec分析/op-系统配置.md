# 系统配置

**目录**

- 系统配置内容（版本0）
  - [`batcherHash`( `bytes32`)](https://github.com/ethereum-optimism/optimism/blob/develop/specs/system_config.md#batcherhash-bytes32)
  - [`overhead`和`scalar`( `uint256,uint256`)](https://github.com/ethereum-optimism/optimism/blob/develop/specs/system_config.md#overhead-and-scalar-uint256uint256)
  - [`gasLimit`( `uint64`)](https://github.com/ethereum-optimism/optimism/blob/develop/specs/system_config.md#gaslimit-uint64)
  - [`unsafeBlockSigner`( `address`)](https://github.com/ethereum-optimism/optimism/blob/develop/specs/system_config.md#unsafeblocksigner-address)
- [编写系统配置](https://github.com/ethereum-optimism/optimism/blob/develop/specs/system_config.md#writing-the-system-config)
- [读取系统配置](https://github.com/ethereum-optimism/optimism/blob/develop/specs/system_config.md#reading-the-system-config)

这`SystemConfig`是 L1 上的合约，可以将汇总配置更改作为日志事件发出。汇总块[派生过程](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md)会获取这些日志事件并应用更改。

## 系统配置内容（版本0）

系统配置合约版本0定义了以下参数：

### `batcherHash`( `bytes32`)

当前授权批处理发送者的版本化哈希，用于作为批处理提交者轮换密钥。第一个字节标识版本。

版本`0`将当前批次提交者以太坊地址 ( `bytes20`) 嵌入版本化哈希的最后 20 个字节中。

将来，此版本化哈希可能会成为对更广泛配置的承诺，以实现更广泛的冗余和/或轮换配置。

### `overhead`和`scalar`( `uint256,uint256`)

L1 费用参数，也称为 Gas Price Oracle (GPO) 参数，会同时更新，并将新的 L1 成本应用于 L2 交易。

### `gasLimit`( `uint64`)

L2 块的 Gas 限制通过系统配置进行配置。L2 气体限制的更改完全应用于引入更改的 L1 源的第一个 L2 块，而不是像 L1 块的限制更新中所示的针对目标的 1/1024 调整。

### `unsafeBlockSigner`( `address`)

区块在 L1 上可用之前，会在 p2p 网络中传播。为了防止 p2p 层上的拒绝服务，这些不安全块必须使用特定密钥进行签名，才能被接受为“规范”不安全块。该键对应的地址是`unsafeBlockSigner`. 为了确保它的值可以通过存储证明以独立于存储布局的方式获取，它被存储在对应于 的特殊存储槽中 `keccak256("systemconfig.unsafeblocksigner")`。

与其他值不同，`unsafeBlockSigner`唯一的值根据区块链策略运行。它不是共识级别参数。

## 编写系统配置

合约`SystemConfig`对所有写入合约功能进行认证，配置管理可以配置为任何类型的以太坊账户或合约。

写入时，会发出一个事件，以便 L2 系统拾取更改，并且新写入的配置变量的副本保留在 L1 状态中，以便通过 L1 合约进行读取。

## 读取系统配置

Rollup 节点通过根据其过去的 L2 链查找起点来初始化其派生过程：

- 当从 L2 创世开始时，将从汇总链配置中检索初始系统配置。
- 当从现有的 L2 链启动时，先前包含的 L1 块被确定为派生起点，因此可以从引用 L1 块作为 L1 起源的最后一个 L2 块中检索系统配置：
  - `batcherHash`，`overhead`并`scalar`从 L1 区块信息交易中检索。
  - `gasLimit`从 L2 块头中检索。
  - 还可以从 L2 块的其他内容（例如标头）检索其他未来变量。

在为给定的 L1 起始输入准备好初始系统配置后，系统配置将通过处理来自每个新 L1 块的所有接收来更新。

对包含的日志事件进行过滤和处理，如下所示：

- 日志事件合约地址必须与汇总`SystemConfig`部署匹配

- 第一个日志事件主题必须与 ABI 哈希匹配`ConfigUpdate(uint256,uint8,bytes)`

- 第二个主题决定版本。未知版本是严重的推导错误。

- 第三个主题确定更新的类型。未知类型是严重的推导错误。

- 剩余的事件数据是不透明的，编码为ABI字节（即包括偏移量和长度数据），并对配置更新进行编码。版本中

  ```
  0
  ```

  支持以下类型：

  - type `0`：`batcherHash`覆盖，作为`bytes32`有效负载。
  - 输入`1`:`overhead`并`scalar`覆盖，作为两个打包`uint256`条目。
  - type `2`：`gasLimit`覆盖，作为`uint64`有效负载。
  - type `3`：`unsafeBlockSigner`覆盖，作为`address`有效负载。

请注意，各个推导阶段可能正在处理不同的 L1 块，因此应维护各个系统配置副本，并在该阶段遍历到下一个 L1 块时应用基于事件的更改。