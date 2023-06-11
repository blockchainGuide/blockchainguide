哈希时间锁定合约（HTLC，Hashed Timelock Contract）是一种特殊的智能合约，用于实现跨链原子交换、支付通道网络和多跳支付等场景。HTLC将支付与哈希锁和时间锁关联起来。哈希锁确保支付在预先设定的条件下进行，而时间锁确保支付在一定时间内完成。

以下是一个基于以太坊的HTLC合约实现的示例。请注意，此代码仅用于演示目的，并未经过全面审计。在实际生产环境中部署合约时，请务必对代码进行彻底审查并进行严格测试。

```solidity
pragma solidity ^0.8.0;

contract HashedTimelock {
    struct LockContract {
        address payable sender;
        address payable recipient;
        uint256 amount;
        uint256 releaseTime;
        bytes32 hashlock;
        bool refunded;
        bool withdrawn;
    }

    mapping(bytes32 => LockContract) public contracts;

    event LogHTLCNew(
        bytes32 indexed contractId,
        address indexed sender,
        address indexed recipient,
        uint256 amount,
        uint256 releaseTime,
        bytes32 hashlock
    );
    event LogHTLCWithdraw(bytes32 indexed contractId);
    event LogHTLCRefund(bytes32 indexed contractId);

    function newContract(
        address payable _recipient,
        bytes32 _hashlock,
        uint256 _timelock
    ) external payable {
        bytes32 contractId = keccak256(
            abi.encodePacked(msg.sender, _recipient, msg.value, _hashlock, block.timestamp)
        );

        require(contracts[contractId].releaseTime == 0, "Contract already exists");

        contracts[contractId] = LockContract(
            msg.sender,
            _recipient,
            msg.value,
            block.timestamp + _timelock,
            _hashlock,
            false,
            false
        );

        emit LogHTLCNew(contractId, msg.sender, _recipient, msg.value, block.timestamp + _timelock, _hashlock);
    }

    function withdraw(bytes32 _contractId, bytes32 _preimage) external {
        LockContract storage c = contracts[_contractId];

        require(c.recipient == msg.sender, "Invalid recipient");
        require(c.withdrawn == false, "Already withdrawn");
        require(sha256(abi.encodePacked(_preimage)) == c.hashlock, "Invalid preimage");
        require(block.timestamp <= c.releaseTime, "Timelock has expired");

        c.withdrawn = true;
        c.recipient.transfer(c.amount);

        emit LogHTLCWithdraw(_contractId);
    }

    function refund(bytes32 _contractId) external {
        LockContract storage c = contracts[_contractId];

        require(c.sender == msg.sender, "Invalid sender");
        require(c.refunded == false, "Already refunded");
        require(c.withdrawn == false, "Already withdrawn");
        require(block.timestamp > c.releaseTime, "Timelock has not expired");

        c.refunded = true;
        c.sender.transfer(c.amount);

        emit LogHTLCRefund(_contractId);
    }
}
```

以下是代码注释，详细说明了合约的功能和结构：

- `LockContract`结构体：定义HTLC合约的数据结构，包括发送方、接收方、金额、释放时间、哈希锁、是否已退款和是否已提现等字段。

- `contracts`映射：存储所有HTLC合约实例，以合约ID为键。

- 事件定义：`LogHTLCNew`、`LogHTLCWithdraw`和`LogHTLCRefund`分别用于记录合约创建、提现和退款操作。

- `newContract`函数：用于创建新的HTLC合约。此函数接受以下参数：
  - `_recipient`：接收方地址。
  - `_hashlock`：哈希锁，为预先定义的哈希值。
  - `_timelock`：时间锁，用于设置合约释放时间。
  函数首先根据输入参数计算合约ID，然后检查该合约是否已存在。如果不存在，则创建并存储新的合约实例，并触发`LogHTLCNew`事件。

- `withdraw`函数：用于从HTLC合约中提现资金。此函数接受以下参数：
  - `_contractId`：要提现的合约ID。
  - `_preimage`：用于解锁哈希锁的原像（解锁值）。
  函数首先检查调用者是否为合约接收方、合约是否已提现、提供的原像是否正确以及是否在时间锁范围内。如果满足这些条件，将资金转移到接收方地址，更新合约状态，并触发`LogHTLCWithdraw`事件。

- `refund`函数：用于退款HTLC合约中的资金。此函数接受以下参数：
  - `_contractId`：要退款的合约ID。
  函数首先检查调用者是否为合约发送方、合约是否已退款、合约是否已提现以及是否超过时间锁。如果满足这些条件，将资金转移到发送方地址，更新合约状态，并触发`LogHTLCRefund`事件。
