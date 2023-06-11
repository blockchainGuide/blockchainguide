> 基于ee547b17853e71ed4e0101ccfd52e70d5acded58
>
> [Uniswap/v2-periphery: 🎚 Peripheral smart contracts for interacting with Uniswap V2 (github.com)](https://github.com/Uniswap/v2-periphery)
>
> [Uniswap/v2-core: 🎛 Core smart contracts of Uniswap V2 (github.com)](



## 概述

UniswapV2Factory合约用来创建资金池，其主要有以下几方面内容：

- 如何创建交易对
- 如何管理收费地址
- 如何查询交易对信息

-----

## 创建交易对

通过工厂模式来创建新的资金池合约，并返回资金池地址。每个交易对都会有相应的合约。

```solidity
   function createPair(address tokenA, address tokenB) external returns (address pair) {
        // 要求 tokenA 和 tokenB 地址不相等
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        
        // 根据地址大小排序，确保唯一性
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
        
        // 要求交易对不存在
        require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); 
        
        // 获取 UniswapV2Pair 合约的字节码
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        
        // 使用参数 token0, token1 计算 salt
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        assembly {
        		// 使用 create2 部署 Pair 合约
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
        
        // Pair 合约初始化
        IUniswapV2Pair(pair).initialize(token0, token1);
        
        // 记录 token0,token1 创建的资金池地址是 pair 
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // 以相反顺序也存储交易对地址，方便查询
        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length);
    }
```



## 管理收费地址

```solidity
    function setFeeTo(address _feeTo) external {
        require(msg.sender == feeToSetter, 'UniswapV2: FORBIDDEN');
        feeTo = _feeTo;
    }

    function setFeeToSetter(address _feeToSetter) external {
        require(msg.sender == feeToSetter, 'UniswapV2: FORBIDDEN');
        feeToSetter = _feeToSetter;
    }
```

`feeTo` 和 `feeToSetter` 是两个与交易手续费相关的地址：

1. `feeTo`：这是接收交易手续费的地址。在 Uniswap V2 中，每次发生交易时，会产生一定比例的手续费。这些手续费的一部分会作为激励分配给流动性提供者，另一部分会发送给 `feeTo` 地址。通常，这个地址会被设置为一个特定的项目或组织（用于支持项目发展、激励生态系统）

2. `feeToSetter`：这是一个具有权限的地址，负责设置 `feeTo` 地址。`feeToSetter` 可以调用 `setFeeTo()` 函数来更改 `feeTo` 地址。在 `UniswapV2Factory` 合约的构造函数中，`feeToSetter` 默认被设置为合约部署者的地址（一般团队控制）。此外，`feeToSetter` 也可以调用 `setFeeToSetter()` 函数来转移这个权限给另一个地址。

之所以只允许 `feeToSetter` 来调用这些函数，是为了确保对手续费分配的控制权只能由具有相应权限的地址进行操作。这样可以防止恶意用户修改手续费接收地址或滥用权限。同时，这种设计也允许项目方或组织方灵活地设置和更改手续费接收地址，以满足不同的需求。