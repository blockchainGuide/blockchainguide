# 存款

[存入交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposited)，也称为[存款](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposits)，是在L1上发起并在L2上执行的交易。本文件概述了一种新的存款[交易类型](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#transaction-type)。它还描述了如何在 L1 上发起存款，以及 L2 上的授权和验证条件。

**词汇注释**：*存入交易*特指L2交易，而 *存款*可以指各个阶段的交易（例如存入L1时）。

**目录**

- 充值交易类型
  - [源哈希计算](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#source-hash-computation)
  - [充值交易种类](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#kinds-of-deposited-transactions)
  - [存入交易的验证和授权](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#validation-and-authorization-of-deposited-transactions)
  - 执行
    - [随机数处理](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#nonce-handling)
- [存款收据](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#deposit-receipt)
- [L1属性充值交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-deposited-transaction)
- L2 上的特殊账户
  - [L1 属性存款人账户](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-depositor-account)
  - L1属性预部署合约
    - [L1属性预部署合约：参考实现](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-predeployed-contract-reference-implementation)
- 用户存入交易
  - 存款合约
    - [地址别名](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#address-aliasing)
    - [存款合约实施：乐观门户](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#deposit-contract-implementation-optimism-portal)

## 充值交易类型

[存入交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposited)与现有交易类型有以下显着区别：

1. 它们源自第 1 层块，并且必须作为协议的一部分包含在内。
2. 它们不包括签名验证（请参阅[用户存入交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#user-deposited-transactions) 了解原理）。
3. 他们在 L1 上购买 L2 Gas，因此 L2 Gas 不可退还。

我们定义了一个新的[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)兼容交易类型，其前缀`0x7E`代表存款交易。

存款具有以下字段（rlp 按照它们在此处出现的顺序进行编码）：

- `bytes32 sourceHash`：源哈希，唯一标识存款的来源。

- `address from`：发件人帐户的地址。

- `address to`：接收者帐户的地址，如果存入的交易是合约创建，则为空（零长度）地址。

- `uint256 mint`：在 L2 上铸造的 ETH 价值。

- `uint256 value`：发送到接收者账户的 ETH 值。

- `uint64 gas`：L2 交易的 Gas 限制。

- ```
  bool isSystemTx
  ```

  ：如果为 true，则交易不会与 L2 区块气池交互。

  - `false`注意：从 Regolith 升级开始，布尔值被禁用（强制为）。

- `bytes data`：通话数据。

[与EIP-155](https://eips.ethereum.org/EIPS/eip-155)交易相比，此交易类型：

- 不包括 a 

  ```
  nonce
  ```

  ，因为它是由 标识的

  ```
  sourceHash
  ```

  。API 响应仍然包含一个

  ```
  nonce
  ```

  属性：

  - 在风化层之前：`nonce`始终是`0`
  - 对于Regolith：`nonce`设置为`depositNonce`相应交易收据的属性。

- 不包含签名信息，并`from`明确地址。API 响应包含归零签名`v`, `r`,`s`值以实现向后兼容。

- 包括新的`sourceHash`、`from`、`mint`和`isSystemTx`属性。API 响应包含这些作为附加字段。

我们选择是`0x7E`因为当前允许交易类型标识符达到`0x7F`。选择高标识符可以最大限度地降低该标识符将来被 L1 链上的另一种交易类型占用的风险。我们不会选择`0x7F`它自己，以防它被用于可变长度编码方案。

### 源哈希计算

存款交易的`sourceHash`是根据来源计算的：

- 用户存入： `keccak256(bytes32(uint256(0)), keccak256(l1BlockHash, bytes32(uint256(l1LogIndex))))`. 其中`l1BlockHash`、 和`l1LogIndex`all 指的是L1上包含充值日志事件。 `l1LogIndex`是该区块的日志事件组合列表中的存款事件日志的索引。
- 存入L1属性： `keccak256(bytes32(uint256(1)), keccak256(l1BlockHash, bytes32(uint256(seqNumber))))`。其中`l1BlockHash`指的是存放信息属性的L1块哈希。且`seqNumber = l2BlockNum - l2EpochStartBlockNum`，其中`l2BlockNum`是存款 tx 包含在 L2 中的 L2 区块号，`l2EpochStartBlockNum`是该纪元中第一个 L2 区块的 L2 区块号。

如果没有`sourceHash`存款，两个不同的存款交易可能具有相同的哈希值。

外部`keccak256`对域中的实际唯一标识信息进行哈希处理，以避免不同类型的源之间的冲突。

我们不使用发送者的随机数来确保唯一性，因为这需要 在块派生期间从[执行引擎读取额外的 L2 EVM 状态。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#execution-engine)

### 充值交易种类

虽然我们只定义了一种新的交易类型，但我们可以根据两种存入交易在 L2 区块中的定位来区分它们：

1. 第一个交易必须是[L1 属性存入交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-deposited-transaction)，后面是
2. 提交到 L1 上的存款馈送合约的一组零个或多个[用户存款交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#user-deposited-transactions)`OptimismPortal`（称为）。用户存入的交易仅存在于 L2 纪元的第一个区块中。

我们仅定义一个新的交易类型，以便最大限度地减少对 L1 客户端软件的修改以及总体复杂性。

### 存入交易的验证和授权

如上所述，存入的交易类型不包括用于验证的签名。相反，授权是由[L2 链派生](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#L2-chain-derivation)过程处理的，正确应用时，只会派生出具有[L1 存款合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#deposit-contract)`from`日志证明的地址的交易。

### 执行

为了执行存入交易：

首先，账户余额`from`必须增加 的金额`mint`。这是无条件的，并且不会在存款失败时恢复。

然后，根据交易的属性初始化存入交易的执行环境，其方式与 EIP-155 交易完全相同。

存款交易的处理方式与类型 3 (EIP-1559) 交易完全相同，但以下情况除外：

- 没有验证任何费用字段：押金没有任何费用，因为它支付了 L1 上的 Gas 费用。
- 未验证任何`nonce`字段：存款没有任何字段，它由其唯一标识`sourceHash`。
- 不处理访问列表：存款没有访问列表，因此会像访问列表为空一样进行处理。
- 不检查是否`from`是外部所有者账户 (EOA)：通过 L1 地址屏蔽确保存款不是 EAO，这可能会在未来的 L1 合约部署中发生变化，例如启用类似账户抽象的机制。
- 风化层升级前：
  - 执行输出显示非标准气体使用情况：
    - 如果`isSystemTx`为 false：执行输出表明它使用了`gasLimit`gas。
    - 如果`isSystemTx`为真：执行输出表明它使用了`0`gas。
- 天然气不会以 ETH 形式退还。（要么不退还，要么利用押金的 Gas 价格为 的事实`0`）
- 不收取交易优先费。不向集体费用接收者支付任何费用。
- 不收取 L1 成本费用，因为押金来自 L1，无需作为数据提交回 L1。
- 不收取基本费用。基本费用总额核算不变。

请注意，这包括像常规交易一样的合约部署行为，并且 Gas 计量是相同的（除了上面与费用相关的更改之外），包括内在 Gas 的计量。

EVM 执行发出的任何非 EVM 状态转换错误都会以特殊方式处理：

- 它转化为EVM错误：即存款将始终被包含在内，但如果遇到非EVM状态转换错误，其收据将指示失败，例如由于帐户不足而无法转移指定数量的 `value`ETH -平衡。
- 在存款的铸造部分之后，世界状态将回滚到 EVM 处理开始时的状态。
- `nonce`世界状态中的 of增加`from`1，使错误相当于本机 EVM 故障。请注意，之前的`nonce`增量可能在 EVM 处理期间发生，但这将首先回滚。

最后，经过上述处理后，执行后处理的运行方式相同：即气池和收据的处理与常规交易相同。然而，从 Regolith 升级开始，存款交易的接收会增加一个附加值 `depositNonce`，存储EVM 处理*之前*注册的发送者`nonce`的值。`from`

请注意，执行输出所述使用的 Gas 会从 Gas 池中减去，但此执行输出值在 Regolith 升级之前具有特殊的边缘情况。

应用程序开发人员请注意：因为`CALLER`和`ORIGIN`被设置为`from`，所以在存款交易期间使用检查的语义`tx.origin == msg.sender`将无法确定调用者是否是 EOA。相反，该支票只能用于识别 L2 存款交易中的第一次调用。然而，此检查仍然满足开发人员使用此检查来确保在`CALLER`调用之前和之后无法执行代码的常见情况。

#### 随机数处理

尽管缺乏签名验证，我们仍然`from`在执行存款交易时增加帐户的随机数。在仅存款汇总的情况下，这对于交易排序或防止重放来说不是必需的，但它与[合约创建](https://github.com/ethereum/execution-specs/blob/617903a8f8d7b50cf71bf1aa733c37897c8d75c1/src/ethereum/frontier/utils/address.py#L40)期间随机数的使用保持了一致性。它还可以简化与下游工具（例如钱包和区块浏览器）的集成。

## 存款收据

交易收据使用符合[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)的标准类型。存款交易收据类型与常规收据相同，但扩展了一个可选`depositNonce`字段。

RLP 编码的共识强制字段是：

- `postStateOrStatus`（标准）：这包含事务状态，请参阅[EIP-658](https://eips.ethereum.org/EIPS/eip-658)。

- ```
  cumulativeGasUsed
  ```

  （标准）：迄今为止区块中使用的天然气，包括本次交易。

  - `CumulativeGasUsed`实际使用的gas是根据与之前交易的差异得出的。
  - 从 Regolith 开始，这说明了存款的实际 Gas 使用量，就像常规交易一样。

- `bloom`（标准）：事务日志的布隆过滤器。

- `logs`（标准）：记录 EVM 处理发出的事件。

- ```
  depositNonce
  ```

  （唯一扩展）：可选字段。存款交易会保留执行期间使用的随机数。

  - 在 Regolith 之前，`depositNonce`必须始终忽略此字段。
  - 对于风化层，`depositNonce`必须始终包含该字段。

从 Regolith 开始，收据 API 响应利用收据更改来获得更准确的响应数据：

- 包含`depositNonce`在 API 响应中的收据 JSON 数据中
- 对于合约部署（当 时`to == null`），`depositNonce`有助于导出正确的`contractAddress`元数据，而不是假设随机数为零。
- 这`cumulativeGasUsed`说明了 EVM 处理中计量的实际气体使用量。

## L1属性充值交易

L1[属性押金交易是发送到](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#l1-attributes-deposited-transaction)[L1属性预部署合约的](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-predeployed-contract)押金交易。

该交易必须具有以下值：

1. `from`是（ [L1属性存款人账户](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-depositor-account)`0xdeaddeaddeaddeaddeaddeaddeaddeaddead0001`的地址 ）
2. `to`是（ [L1属性预部署合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-predeployed-contract)`0x4200000000000000000000000000000000000015`的地址）。
3. `mint`是`0`
4. `value`是`0`
5. `gasLimit`设置为 150,000,000。
6. `isSystemTx`设置为`true`.
7. `data`是对[L1 属性预部署合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-predeployed-contract)函数 的[ABI](https://docs.soliditylang.org/en/v0.8.10/abi-spec.html)编码调用，具有与相应 L1 块关联的正确值（参见 [参考实现](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-predeployed-contract-reference-implementation)）。`setL1BlockValues()`

如果风化层升级处于活动状态，某些字段将被覆盖：

1. `gasLimit`设置为 1,000,000
2. `isSystemTx`被设定为`false`

系统发起的 L1 属性交易不会为其分配的任何 ETH 收取费用`gasLimit`，因为它实际上是状态转换处理的一部分。

## L2 上的特殊账户

L1属性存款交易涉及两个特殊目的账户：

1. L1属性存款人账户
2. L1属性预部署合约

### L1 属性存款人账户

存款人账户是一个没有已知私钥的[EOA 。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#eoa)它有地址 `0xdeaddeaddeaddeaddeaddeaddeaddeaddead0001`。 在执行 L1 属性存入交易期间，其值由`CALLER`和操作码返回。`ORIGIN`

### L1属性预部署合约

L2 上地址 处的预部署合约`0x4200000000000000000000000000000000000015`，保存存储中相应 L1 区块中的某些区块变量，以便在执行后续存入交易时可以访问它们。

预部署存储以下值：

- L1块属性：

  - `number`( `uint64`)
  - `timestamp`( `uint64`)
  - `basefee`( `uint256`)
  - `hash`( `bytes32`)

- `sequenceNumber`( `uint64`)：这等于相对于纪元开始的 L2 区块编号，即 L1 属性上次更改的 L2 区块到 L2 区块高度的距离，并在新纪元开始时重置为 0。

- 与 L1 块相关的系统配置，请参阅

  系统配置规范

  ：

  - `batcherHash`( `bytes32`)：对当前正在运行的批次提交者的版本化承诺。
  - `overhead`( `uint256`)：应用于此 L2 区块中交易的 L1 成本计算的 L1 费用开销。
  - `scalar`( `uint256`)：应用于此 L2 区块中交易的 L1 成本计算的 L1 费用标量。

该合约实施授权方案，使其仅接受来自[存款人账户的](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#l1-attributes-depositor-account)状态更改调用。

合约具有如下solidity接口，可以按照 [合约ABI规范](https://docs.soliditylang.org/en/v0.8.10/abi-spec.html)进行交互。

#### L1属性预部署合约：参考实现

L1 属性预部署合约的参考实现可以在[L1Block.sol](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L2/L1Block.sol)中找到。

`pnpm build`在目录中运行后`packages/contracts`，要添加到创世文件的字节码将位于`deployedBytecode`构建工件文件的字段 中，位置为`/packages/contracts/artifacts/contracts/L2/L1Block.sol/L1Block.json`.

## 用户存入交易

[用户存入交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#user-deposited-transaction)是由[L2链衍生](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#L2-chain-derivation)过程 生成的[存入交易](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#the-deposited-transaction-type)。每笔用户充值交易的内容由L1上的[充值合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#deposit-contract)发出的 相应事件决定。`TransactionDeposited`

1. `from`与发出的值没有变化（尽管它可能已转换为`OptimismPortal`存款馈送合约中的别名）。

2. ```
   to
   ```

   是任意20字节地址（包括零地址）

   - 如果创建合约（参见`isCreation`），该地址设置为`null`。

3. `mint`设置为发射值。

4. `value`设置为发射值。

5. `gaslimit`与发射值相比没有变化。它必须至少为 21000。

6. `isCreation``true`如果交易是合约创建，则设置为，`false`否则。

7. `data`与发射值相比没有变化。根据它的值，`isCreation`它被处理为调用数据或合约初始化代码。

8. `isSystemTx`由汇总节点为某些具有不计量执行的事务设置。用于`false`用户存入交易

### 存款合约

存款合约部署到L1。存款交易源自`TransactionDeposited`存款合约发出的事件中的值。

保证金合约负责维护[保证 Gas 市场](https://github.com/ethereum-optimism/optimism/blob/develop/specs/guaranteed-gas-market.md)，对 L2 上使用的 Gas 收取保证金，并保证单个 L1 区块的保证 Gas 总量不超过 L2 区块 Gas 限额。

存款合约处理两种特殊情况：

1. 合约创建押金，通过将`isCreation`标志设置为 来表示`true`。如果`to`地址非零，合约将恢复。
2. 来自合约账户的调用，在这种情况下，该`from`值将转换为其 L2 [别名](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#address-aliasing)。

#### 地址别名

如果调用者是合约，则地址将通过添加 `0x1111000000000000000000000000000000001111`来转换。数学是`unchecked`在 Solidity 上完成的`uint160`，因此该值会溢出。这可以防止 L1 上的合约与 L2 上的合约具有相同地址但代码不同的攻击。我们可以安全地忽略 EOA 的这一点，因为它们保证具有相同的“代码”（即根本没有代码）。这也使得用户即使在 Sequencer 关闭时也可以与 L2 上的合约进行交互。

#### 存款合约实施：乐观门户

存款合约的参考实现可以在[OptimismPortal.sol](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/contracts/L1/OptimismPortal.sol)中找到。