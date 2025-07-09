# 使用CREATE2部署智能合约

## 简介

CREATE2 是 Ethereum EVM 中的一种操作码，用于部署智能合约。与 CREATE 操作码不同，CREATE2 的独特之处在于它允许在部署合约之前预测合约地址。这种特性为开发者提供了新的部署方法和更多的灵活性，特别是在需要预测合约地址的场景下。

## 创建合约地址的原理

在 Ethereum 中，合约地址是通过部署者地址和该地址的随机数（`nonce`）计算得出的。CREATE2 改变了这种计算方式，通过部署者地址、一个盐值（`salt`）和合约的字节码进行计算。这意味着，只要部署者、盐值和字节码不变，合约地址将始终相同。

计算公式如下：

```
keccak256( 0xff ++ address ++ salt ++ keccak256(init_code))[12:]
```

## 使用案例

### 1. 预测合约地址

由于 CREATE2 计算合约地址的方式，它可以让开发者在部署合约之前知道合约地址。这种特性对于某些去中心化应用（如闪电网络和状态通道）非常有用。

### 2. 惰性部署

开发者可以在需要时才部署合约，而在此之前，他们可以预先告知用户合约的地址。这样一来，用户可以在合约实际部署之前与其交互，从而节省部署成本。

### 3. 代理合约模式

在某些场景下，开发者可能希望部署多个功能相似的合约。使用 CREATE2 可以创建一个固定地址的代理合约，从而使得所有相关合约共享相同的接口。

## 如何使用 CREATE2

以下是一个使用 Solidity 实现的简单示例，演示了如何使用 CREATE2 部署合约。

```solidity
pragma solidity ^0.8.0;

contract DeployedContract {
    uint256 public value;

    constructor(uint256 _value) {
        value = _value;
    }
}

contract Factory {
    function deploy(uint256 _value, bytes32 _salt) public {
        bytes memory bytecode = type(DeployedContract).creationCode;
        bytes32 bytecodeHash = keccak256(abi.encodePacked(bytecode, _value));

        address addr;
        assembly {
            addr := create2(0, add(bytecode, 32), mload(bytecode), _salt)
        }
        
        require(addr != address(0), "Failed to deploy contract");
    }

    function predictAddress(uint256 _value, bytes32 _salt) public view returns (address) {
        bytes memory bytecode = type(DeployedContract).creationCode;
        bytes32 bytecodeHash = keccak256(abi.encodePacked(bytecode, _value));
        
        return address(uint160(uint256(keccak256(abi.encodePacked(
            byte(0xff),
            address(this),
            _salt,
            bytecodeHash
        )))));
    }
}

```



## 场景

### 灵活部署合约

