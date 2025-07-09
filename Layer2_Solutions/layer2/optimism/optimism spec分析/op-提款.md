# 提款

[提款](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#withdrawal)是跨域交易，在 L2 上发起，并由在 L1 上执行的交易完成。值得注意的是，L2 账户可以使用提款来调用 L1 合约，或者将 ETH 从 L2 账户转移到 L1 账户。

**词汇注释**：*提现*可以指交易过程中各个阶段的交易，但我们引入更具体的术语来区分：

- 提款*发起交易*特指 L2 上发送到提款预部署的交易。
- 提*现证明交易*特指证明提现正确（已包含在根在L1上的merkle树中）的L1交易。
- 提*现完成交易*特指完成并中继提现的L1交易。

提款是通过调用 Message Passer 预部署合约在 L2 上发起的，该合约在其存储中记录了消息的重要属性。通过调用 来在 L1 上证明提款`OptimismPortal`，这证明包含此提款消息。提款是通过调用合约在 L1 上完成的`OptimismPortal`，该合约验证自提款消息被证明以来故障挑战期已经过去。

这样，提款与存款不同，[存款在](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#deposits)[执行引擎](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#execution-engine)客户端中使用特殊的交易类型 。相反，提款交易必须使用 L1 上的智能合约来完成。

**目录**

- 提现流程
  - [在 L2 上](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md#on-l2)
  - [位于 L1](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md#on-l1)
- L2ToL1MessagePasser 合约
  - [提款时地址不会使用别名](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md#addresses-are-not-aliased-on-withdrawals)
- [乐观门户合约](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md#the-optimism-portal-contract)
- [提款验证和最终确定](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md#withdrawal-verification-and-finalization)
- 安全考虑
  - [提款验证的关键属性](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md#key-properties-of-withdrawal-verification)
  - [处理已成功验证但中继失败的消息](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md#handling-successfully-verified-messages-that-fail-when-relayed)

## 提现流程

我们首先描述发起和完成提款的端到端流程：

### 在 L2 上

L2 账户向`L2ToL1MessagePasser`预部署合约发送一条提款消息（也可能是 ETH）。这是一个非常简单的合约，存储提款数据的哈希值。

### 位于 L1

1. 中继[者](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#withdrawals)提交提款证明交易以及合约所需的输入`OptimismPortal`。中继者不一定是在 L2 上发起撤回的同一实体。这些输入包括提款交易数据、包含证明和区块号。区块号必须是存在 L2 输出根的区块号，该根号承诺在 L2 上注册的提款。
2. 合约从的 函数`OptimismPortal`中检索给定块号的输出根，并在内部执行其余的验证过程。`L2OutputOracle``getL2Output()`
3. 如果证明验证失败，则调用恢复。否则，哈希值将被记录以防止其被重新证明。请注意，如果相应的输出根发生变化，则可以多次证明撤回。
4. 提现被证明后，进入 7 天的挑战期，让其他网络参与者有时间挑战相应输出根的完整性。
5. 一旦挑战期结束，中继者就会向 `OptimismPortal`合约提交提款完成交易。中继者不需要是在 L2 上发起提款的同一实体。
6. 合约`OptimismPortal`接收提现交易数据并验证提现是否已被证明并通过挑战期。
7. 如果不满足要求，呼叫将恢复。否则，呼叫将被转发，并记录哈希值以防止重播。

## L2ToL1MessagePasser 合约

通过调用 L2ToL1MessagePasser 合约的函数来发起提款`initiateWithdrawal`。L2ToL1MessagePasser 是一个简单的预部署合约，用于`0x4200000000000000000000000000000000000016` 存储要撤回的消息。

```
interface L2ToL1MessagePasser {
    event MessagePassed(
        uint256 indexed nonce, // this is a global nonce value for all withdrawal messages
        address indexed sender,
        address indexed target,
        uint256 value,
        uint256 gasLimit,
        bytes data,
        bytes32 withdrawalHash
    );

    event WithdrawerBalanceBurnt(uint256 indexed amount);

    function burn() external;

    function initiateWithdrawal(address _target, uint256 _gasLimit, bytes memory _data) payable external;

    function messageNonce() public view returns (uint256);

    function sentMessages(bytes32) view external returns (bool);
}
```



该`MessagePassed`事件包括散列并存储在映射中的所有数据`sentMessages`以及散列本身。

### 提款时地址不会使用别名

当合约进行存款时，发送者的地址是[别名](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#address-aliasing)。提款则不然，提款不会修改发件人的地址。不同之处在于：

- 在 L2 上，存款发送者的地址由操作码返回`CALLER`，这意味着合约无法轻易判断调用是源自 L1 还是 L2，而
- `l2Sender()`在L1上，通过调用合约上的函数来访问提款发送者的地址`OptimismPortal` 。

呼叫`l2Sender()`消除了有关呼叫源自哪个域的任何歧义。尽管如此，开发人员需要认识到，拥有相同的地址并不意味着 L2 上的合约将与 L1 上的合约表现相同。

## 乐观门户合约

Optimism Portal 充当 Optimism L2 的入口和出口点。它是一个继承自[OptimismPortal](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md#deposit-contract)合约的合约，此外还提供了以下提现接口：

- [`WithdrawalTransaction`类型](https://github.com/ethereum-optimism/optimism/blob/6c6d142d7bb95faa11066aab5d8aed7187abfe38/packages/contracts-bedrock/contracts/libraries/Types.sol#L76-L83)
- [`OutputRootProof`类型](https://github.com/ethereum-optimism/optimism/blob/6c6d142d7bb95faa11066aab5d8aed7187abfe38/packages/contracts-bedrock/contracts/libraries/Types.sol#L33-L38)

```
interface OptimismPortal {

    event WithdrawalFinalized(bytes32 indexed withdrawalHash, bool success);


    function l2Sender() returns(address) external;

    function proveWithdrawalTransaction(
        Types.WithdrawalTransaction memory _tx,
        uint256 _l2OutputIndex,
        Types.OutputRootProof calldata _outputRootProof,
        bytes[] calldata _withdrawalProof
    ) external;

    function finalizeWithdrawalTransaction(
        Types.WithdrawalTransaction memory _tx
    ) external;
}
```



## 提款验证和最终确定

需要提供以下信息来证明并最终确定提款：

- 提现交易数据：
  - `nonce`：所提供消息的随机数。
  - `sender`：L2 上的消息发送者地址。
  - `target`：L1 上的目标地址。
  - `value`：发送到目标的 ETH。
  - `data`：要发送到目标的数据。
  - `gasLimit`：要转发到目标的气体。
- 证明和验证数据：
  - `l2OutputIndex`：L2 输出中的索引，可以在其中找到适用的输出根。
  - `outputRootProof``bytes32`：用于导出输出根的四个值。
  - `withdrawalProof`：L2ToL1MessagePasser 合约中给定提款的包含证明。

这些输入必须满足以下条件：

1. 必须`l2OutputIndex`是 L2 输出中包含适用输出根的索引。
2. `L2OutputOracle.getL2Output(l2OutputIndex)`返回一个非零值`OutputProposal`。
3. 值的 keccak256 哈希值`outputRootProof`等于`outputRoot`.
4. 这`withdrawalProof`是一个有效的包含证明，证明提款交易数据的哈希值包含在 L2 上的 L2ToL1MessagePasser 合约的存储中。

## 安全考虑

### 提款验证的关键属性

1. 不应“双花”提款，即。在 L1 上中继与 L2 上发起的消息不对应的撤回。作为参考，请参阅Polygon 上发现的此类漏洞的[文章。](https://gerhard-wagner.medium.com/double-spending-bug-in-polygons-plasma-bridge-2e0954ccadf1)
2. 对于 L2 上发起的每次提款（即具有唯一的`messageNonce()`），必须满足以下属性：
   1. 除非提款的outputRoot已经改变，否则只能证明一次提款。
   2. 应该只能完成一次提款。
   3. 应该不可能在任何字段被修改的情况下中继消息，即。
      1. 修改该`sender`字段将启用“欺骗”攻击。
      2. 修改`target`、`data`或`value`字段将使攻击者能够危险地改变提款的预期结果。
      3. 修改`gasLimit`可能会使中继的成本过高，或者允许中继器导致`target`.

### 处理已成功验证但中继失败的消息

如果合约中中继调用的执行失败`target`，不幸的是无法确定它是否“应该”失败，以及它是否应该“可重放”。因此，为了最大限度地降低复杂性，我们没有提供任何重播功能，如果需要，可以在外部公用事业合同中实现。