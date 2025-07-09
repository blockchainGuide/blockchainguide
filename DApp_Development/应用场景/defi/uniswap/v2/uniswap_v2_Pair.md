> 基于ee547b17853e71ed4e0101ccfd52e70d5acded58
>
> [Uniswap/v2-periphery: 🎚 Peripheral smart contracts for interacting with Uniswap V2 (github.com)](https://github.com/Uniswap/v2-periphery)
>
> [Uniswap/v2-core: 🎛 Core smart contracts of Uniswap V2 (github.com)](https://github.com/Uniswap/v2-core)



## 初始化

```solidity
  constructor() public {
        factory = msg.sender;
    }

    // called once by the factory at time of deployment
    function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, 'UniswapV2: FORBIDDEN'); // sufficient check
        token0 = _token0;
        token1 = _token1;
    }
```

- Pair 合约只能是UniswapV2Factory 合约来初始化， 且只会调用一次



## 添加流动性并发行LP

当用户希望向 Uniswap V2 的流动性池中添加流动性时，他们会将代币发送到某个流动性池对应的 UniswapV2Pair 合约地址，然后通过外部智能合约或 dApp 触发 `mint` 函数，将代币转入流动性池并生成相应的流动性代币。下面逐行解析：

1. 函数定义与锁定：
```solidity
function mint(address to) external lock returns (uint liquidity) {
```
`mint` 函数接收一个 `to` 参数，表示接收新生成的流动性代币的地址。`lock` 修饰符确保在函数执行期间合约被锁定，防止重入攻击。

2. 获取当前储备量：
```solidity
(uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
```
调用 `getReserves` 函数获取当前的储备量，用于后续计算。

3. 获取合约内部的代币余额：
```solidity
uint balance0 = IERC20(token0).balanceOf(address(this));
uint balance1 = IERC20(token1).balanceOf(address(this));
```
查询合约内部 token0 和 token1 的余额。

4. 计算添加的流动性金额：
```solidity
uint amount0 = balance0.sub(_reserve0);
uint amount1 = balance1.sub(_reserve1);
```
计算用户实际添加的流动性金额，即合约内部代币余额与当前储备量的差值。

5. 调用 `_mintFee` 函数：
```solidity
bool feeOn = _mintFee(_reserve0, _reserve1);
```
调用 `_mintFee` 函数，处理手续费收益，并获取手续费是否开启的状态。

6. 计算流动性代币数量：
```solidity
uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
if (_totalSupply == 0) {
    liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
   _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
} else {
    liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
}
```
基于当前的总供应量（totalSupply）和添加的流动性金额，计算用户应该获得的流动性代币数量。如果当前总供应量为零，则生成的流动性代币数量为两种代币金额乘积的平方根减去最小流动性。同时，将最小流动性代币永久锁定。如果当前总供应量不为零，则生成的流动性代币数量取两种计算方法中的较小值。

7. 确保生成的流动性代币数量大于零：
```solidity
require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
```
检查生成的流动性代币数量是否大于零。如果不是，则抛出异常。

8. 发放流动性代币：
```solidity
_mint(to, liquidity);
```
调用内部的 `_mint` 函数，向指定的 `to` 地址发放新生成的流动性代币。

9. 更新储备量和价格累积值：
```solidity
_update(balance0, balance1, _reserve0, _reserve1);
```
调用内部的 `_update` 函数，根据新的代币余额和当前储备量更新储备量以及价格累积值。

10. 更新 kLast：
```solidity
if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
```
如果手续费开启，更新 kLast 的值。kLast 用于计算交易手续费的收益。

11. 触发 Mint 事件：
```solidity
emit Mint(msg.sender, amount0, amount1);
```
触发 Mint 事件，通知外部监听器已成功添加流动性并生成了新的流动性代币。

总结一下，`mint` 函数用于向流动性池中添加流动性并生成流动性代币。它首先获取当前的储备量和合约内部的代币余额，然后根据添加的流动性金额计算用户应获得的流动性代币数量。接着，发放新生成的流动性代币，更新储备量和价格累积值，最后触发 Mint 事件。

## 移除流动性并销毁LP

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function burn(address to) external lock returns (uint amount0, uint amount1) {
        (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
        address _token0 = token0; // gas savings
        address _token1 = token1; // gas savings
        uint balance0 = IERC20(_token0).balanceOf(address(this));
        uint balance1 = IERC20(_token1).balanceOf(address(this));
        uint liquidity = balanceOf[address(this)];

        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        amount0 = liquidity.mul(balance0) / _totalSupply; // using balances ensures pro-rata distribution
        amount1 = liquidity.mul(balance1) / _totalSupply; // using balances ensures pro-rata distribution
        require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
        _burn(address(this), liquidity);
        _safeTransfer(_token0, to, amount0);
        _safeTransfer(_token1, to, amount1);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Burn(msg.sender, amount0, amount1, to);
    }
```

	这段代码是 Uniswap V2 代币配对合约中的 `burn` 函数。该函数的主要目的是销毁流动性代币以移除流动性，并将代币按比例返回给调用者。以下是对该代码段的逐步分析：

1. 从合约中获取储备量（`_reserve0` 和 `_reserve1`），这是一个用于节省 Gas 的优化。

2. 将 `token0` 和 `token1` 存储在局部变量（`_token0` 和 `_token1`）中以进行 Gas 优化。

3. 查询代币0和代币1在当前合约中的余额（`balance0` 和 `balance1`）。

4. 查询当前合约中流动性代币的余额（`liquidity`）。

5. 检查是否需要收取手续费（`feeOn`），如果需要，则调用 `_mintFee` 函数来处理费用。

6. 将总供应量存储在局部变量（`_totalSupply`）中。

7. 根据销毁的流动性代币数量计算要提取的代币0（`amount0`）和代币1（`amount1`）的数量。这是通过使用当前合约中的余额确保按比例分配计算出来的。

8. 确保提取的代币数量（`amount0` 和 `amount1`）大于 0，否则抛出异常。

9. 销毁合约中的流动性代币。

10. 将代币0和代币1安全地转移到指定的接收者地址（`to`）。

11. 再次查询合约中代币0和代币1的余额，以便在后续的 `_update` 函数中使用。

12. 调用 `_update` 函数，用最新的余额更新储备量和时间戳。

13. 如果启用了手续费，将 `kLast` 设置为最新的储备量乘积，以备后续计算手续费时使用。

14. 触发一个 `Burn` 事件，通知外部系统销毁了流动性代币并提取了资产。

总之，这段代码的核心目标是销毁流动性代币并将对应的代币按比例返回给调用者。在此过程中，它还处理了手续费的收取（如果需要）并更新了合约的储备量。

## 代币兑换

```solidity
 function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
        (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

        uint balance0;
        uint balance1;
        {
            // scope for _token{0,1}, avoids stack too deep errors
            address _token0 = token0;
            address _token1 = token1;
            require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
            if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
            if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
            if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
            balance0 = IERC20(_token0).balanceOf(address(this));
            balance1 = IERC20(_token1).balanceOf(address(this));
        }
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        {
            // scope for reserve{0,1}Adjusted, avoids stack too deep errors
            uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
            uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
            require(
                balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000 ** 2),
                'UniswapV2: K'
            );
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
```

1. 确保至少有一个输出代币的数量大于0，否则抛出异常。

2. 从合约中获取当前的储备量（`_reserve0` 和 `_reserve1`）。

3. 确保请求的输出代币数量小于当前储备量，否则抛出异常。

4. 定义局部变量 `balance0` 和 `balance1`。在这个代码块中，它们还没有被赋值。

5. 创建一个新的作用域以避免栈过深错误。在这个作用域中：

   a. 将 `token0` 和 `token1` 存储在局部变量（`_token0` 和 `_token1`）中。
   
   b. 确保接收者地址（`to`）不是代币合约地址之一。
   
   c. 如果 `amount0Out` 大于 0，则将代币0发送到接收者地址。
   
   d. 如果 `amount1Out` 大于 0，则将代币1发送到接收者地址。
   
   e. 如果 `data` 参数长度大于 0，则调用接收者地址上的 `uniswapV2Call` 函数。这允许接收者合约在交换完成后执行任意逻辑（例如，将交换的代币用于进一步的交易）。
   
   f. 查询合约中代币0和代币1的余额（`balance0` 和 `balance1`）。

6. 计算输入代币的实际数量（`amount0In` 和 `amount1In`）。

7. 确保至少有一个输入代币的数量大于 0，否则抛出异常。

8. 创建一个新的作用域以避免栈过深错误。在这个作用域中：

   a. 计算调整后的余额（`balance0Adjusted` 和 `balance1Adjusted`），通过将余额乘以 1000 减去输入代币数量乘以 3。这是为了考虑交易费用。
   
   b. 检查交易后的调整后余额乘积是否大于或等于交易前储备量乘积的 1000 平方。这是为了确保交易后的状态满足恒定乘积公式，即 K = x * y，其中 x 和 y 分别为两个代币的储备量。

9. 使用新的余额更新储备量和时间戳。

10. 触发一个 `Swap` 事件，通知外部系统已完成代币交换。

​	

## 更新资金池

```solidity
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
	....
}
```

 UniswapV2Pair 合约中的 `_update` 函数，它的作用是在交易、添加或移除流动性等操作之后，更新代币对的储备量（reserve0 和 reserve1）以及价格累积值（price0CumulativeLast 和 price1CumulativeLast）。接下来，我将逐行解释这段代码：

1. 检查余额是否溢出：
```solidity
require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
```
此行代码确保 balance0 和 balance1 的值不会超过 uint112 类型的最大值，防止整数溢出。

2. 获取区块时间戳：
```solidity
uint32 blockTimestamp = uint32(block.timestamp % 2**32);
```
此行代码将当前区块的时间戳转换为 uint32 类型。

3. 计算时间差：
```solidity
uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
```
计算当前区块时间戳和上次更新时间戳之间的时间差。注意，此处的溢出是有意为之，因为 Solidity 中的 uint 类型在溢出时会回绕。

4. 更新价格累积值：
```solidity
if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
    price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
    price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
}
```
如果时间差大于零且两个代币的储备量都不为零，那么更新价格累积值。使用 UQ112x112 库进行定点数计算，计算价格比率并乘以时间差，然后将结果累加到相应的价格累积值上。这些累积值在计算价格预言机时会用到。

5. 更新储备量和区块时间戳：
```solidity
reserve0 = uint112(balance0);
reserve1 = uint112(balance1);
blockTimestampLast = blockTimestamp;
```
将传入的 balance0 和 balance1 分别赋值给 reserve0 和 reserve1，并更新 blockTimestampLast。

6. 触发 Sync 事件：
```solidity
emit Sync(reserve0, reserve1);
```
触发 Sync 事件，通知外部监听器储备量已更新。

这个 `_update` 函数主要用于在交易、添加或移除流动性等操作之后，更新代币对的储备量和价格累积值。这些更新对于保持合约状态和计算价格预言机是必要的。