> 基于ee547b17853e71ed4e0101ccfd52e70d5acded58
>
> [Uniswap/v2-periphery: 🎚 Peripheral smart contracts for interacting with Uniswap V2 (github.com)](https://github.com/Uniswap/v2-periphery)
>
> [Uniswap/v2-core: 🎛 Core smart contracts of Uniswap V2 (github.com)](



Router02 合约是用户使用 Uniswap-v2 进行交换直接调用的合约。用户只会跟router进行交互，而底层的核心合约则由router来调用。

​	

## 添加流动性

```solidity
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
        (amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin);
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA);
        TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
        liquidity = IUniswapV2Pair(pair).mint(to);
    }
    
        function _addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin
    ) internal virtual returns (uint amountA, uint amountB) {
        // create the pair if it doesn't exist yet
        if (IUniswapV2Factory(factory).getPair(tokenA, tokenB) == address(0)) {
            IUniswapV2Factory(factory).createPair(tokenA, tokenB);
        }
        (uint reserveA, uint reserveB) = UniswapV2Library.getReserves(factory, tokenA, tokenB);
        if (reserveA == 0 && reserveB == 0) {
            (amountA, amountB) = (amountADesired, amountBDesired);
        } else {
            uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
            if (amountBOptimal <= amountBDesired) {
                require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
                (amountA, amountB) = (amountADesired, amountBOptimal);
            } else {
                uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
                assert(amountAOptimal <= amountADesired);
                require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
                (amountA, amountB) = (amountAOptimal, amountBDesired);
            }
        }
    }
```



`addLiquidity` 函数是在 Uniswap V2 的路由器合约（UniswapV2Router02）中实现的。它的设计意图是提供一个向流动性池添加流动性的入口。当用户希望向某个流动性池中添加流动性时，他们会通过调用这个函数来实现。

分析这个函数的步骤和设计意图：

1. 参数列表：
   - `address tokenA`：代币 A 的地址；
   - `address tokenB`：代币 B 的地址；
   - `uint amountADesired`：用户希望添加的代币 A 数量；
   - `uint amountBDesired`：用户希望添加的代币 B 数量；
   - `uint amountAMin`：添加流动性后，用户接受的最小代币 A 数量；
   - `uint amountBMin`：添加流动性后，用户接受的最小代币 B 数量；
   - `address to`：接收流动性代币的地址；
   - `uint deadline`：交易的最后期限（Unix 时间戳）。

   `amountBDesired`、`amountAMin` 和 `amountBMin` 这四个参数是为了增加交易的灵活性和安全性。它们的作用如下：
   
   1. `amountADesired` 和 `amountBDesired`：用户希望向流动性池中添加的代币 A 和代币 B 的数量。这两个参数表达了用户的意愿，即用户期望向流动性池中添加多少代币。
   2. `amountAMin` 和 `amountBMin`：添加流动性后，用户接受的最小代币 A 和代币 B 数量。这两个参数作为一个安全阈值，确保用户在添加流动性时不会因市场波动或其他原因而损失过多资产。
   
2. 计算实际添加到流动性池的代币数量：
```solidity
(amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin);
```
调用内部的 `_addLiquidity` 函数，根据用户期望添加的代币数量和最小接受数量，计算实际需要添加到流动性池的代币数量。

3. 获取交易对地址：
```solidity
address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
```
调用 `UniswapV2Library.pairFor` 函数，根据 tokenA 和 tokenB 的地址以及工厂合约地址，获取对应的交易对合约（UniswapV2Pair）地址。

4. 转移代币至交易对合约：
```solidity
TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA);
TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
```
使用 `TransferHelper.safeTransferFrom` 函数，将用户实际添加的代币 A 和代币 B 数量从用户地址（`msg.sender`）转移到交易对合约地址（`pair`）。

5. 调用交易对合约的 `mint` 函数：
```solidity
liquidity = IUniswapV2Pair(pair).mint(to);
```
调用交易对合约（UniswapV2Pair）的 `mint` 函数，将代币添加到流动性池，并为用户生成相应的流动性代币。`liquidity` 变量存储了用户获得的流动性代币数量。

## 移除流动性

```solidity
function removeLiquidity(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) public virtual override ensure(deadline) returns (uint amountA, uint amountB) {
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity); // send liquidity to pair
        (uint amount0, uint amount1) = IUniswapV2Pair(pair).burn(to);
        (address token0, ) = UniswapV2Library.sortTokens(tokenA, tokenB);
        (amountA, amountB) = tokenA == token0 ? (amount0, amount1) : (amount1, amount0);
        require(amountA >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
        require(amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
    }
```



这是 `removeLiquidity` 函数的代码，它允许用户从流动性池中移除流动性，并将相应的代币 A 和代币 B 数量返回给用户。下面是该函数的详细解释：

1. 函数接收以下参数：
   - `tokenA` 和 `tokenB`：交易对中的两个代币地址
   - `liquidity`：用户希望移除的流动性代币数量
   - `amountAMin` 和 `amountBMin`：用户希望获取的最小代币 A 和代币 B 数量
   - `to`：接收代币 A 和代币 B 的地址
   - `deadline`：交易截止时间，用于确保交易在某个时间点之前完成

2. 使用 `UniswapV2Library.pairFor` 函数获取交易对合约（UniswapV2Pair）地址。

3. 调用 `IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity)`，将用户持有的流动性代币发送到交易对合约地址。

4. 调用交易对合约的 `burn` 函数：`IUniswapV2Pair(pair).burn(to)`。`burn` 函数销毁流动性代币，并将相应的代币 A 和代币 B 数量返回给用户。

5. 使用 `UniswapV2Library.sortTokens(tokenA, tokenB)` 确定代币 A 和代币 B 的顺序。

6. 将返回的代币数量赋值给 `amountA` 和 `amountB`。如果 `tokenA` 是 `token0`，则 `amountA = amount0`，`amountB = amount1`。如果 `tokenA` 是 `token1`，则 `amountA = amount1`，`amountB = amount0`。

7. 使用 `require` 语句检查用户实际获得的代币 A 和代币 B 数量是否大于或等于其期望的最小数量。如果不满足条件，则抛出异常。

通过这个函数，用户可以从流动性池中移除流动性，并将相应的代币 A 和代币 B 数量返还给用户。用户可以通过指定期望的最小代币数量来保护自己不受价格波动的影响。

同时v2也支持让用户在移除流动性时无需提前批准合约地址转移流动性代币，而是通过签名的方式来完成这一操作。这样可以节省用户的 Gas 费用，提高交易效率。

## swap

```solidity
    function _swap(uint[] memory amounts, address[] memory path, address _to) internal virtual {
        for (uint i; i < path.length - 1; i++) {
            (address input, address output) = (path[i], path[i + 1]);
            (address token0, ) = UniswapV2Library.sortTokens(input, output);
            uint amountOut = amounts[i + 1];
            (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOut) : (amountOut, uint(0));
            address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
            IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output)).swap(
                amount0Out,
                amount1Out,
                to,
                new bytes(0)
            );
        }
    }

    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts) {
        amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
        require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
        TransferHelper.safeTransferFrom(
            path[0],
            msg.sender,
            UniswapV2Library.pairFor(factory, path[0], path[1]),
            amounts[0]
        );
        _swap(amounts, path, to);
    }
```



在 Uniswap 中，swap 的基本原理是通过自动做市商 (AMM) 模型实现的，交易对为两种代币提供流动性。交易对合约中，每种代币都有一个对应的储备。当用户希望进行代币交换时，他们需要向交易对提供一种代币作为输入（买方），并从交易对获取另一种代币作为输出（卖方）。交易的价格由两种代币的储备量决定，遵循恒定乘积公式 x * y = k，其中 x 和 y 分别代表两种代币的储备量，k 是恒定值。

根据用户的需求，Uniswap 提供了两种类型的 swap：

1. 精确输入、不精确输出：在这种情况下，用户提供一个确切的输入代币数量，并期望得到尽可能多的输出代币。由于交易过程中可能会受到价格滑点的影响，因此输出代币的数量并不是一个精确的值。用户可以设置一个最低可接受的输出代币数量（`amountOutMin`），以防止由于滑点过大而导致的不利交易。
2. 不精确输入、精确输出：在这种情况下，用户希望获得一个确切的输出代币数量，并愿意为此提供尽可能少的输入代币。类似地，由于交易过程中可能会受到价格滑点的影响，因此所需的输入代币数量并不是一个精确的值。用户可以设置一个最大可接受的输入代币数量（`amountInMax`），以防止由于滑点过大而支付过多的输入代币。

通过下面来举例：

`swapTokensForExactTokens`函数用于在 Uniswap V2 交易对中完成一个不精确输入、精确输出的代币兑换。以下是对该函数的详细解释：

1. 函数接收以下参数：
   - `amountOut`：用户希望获得的精确输出代币数量
   
   - `amountInMax`：用户愿意提供的输入代币的最大数量
   
   - `path`：代币交换路径的地址数组，指示交换应该遵循的路径

     `path` 是一个地址数组，表示代币交换过程中应遵循的路径。它由一系列代币地址组成，这些地址按照希望交换的顺序排列。数组中的第一个地址代表要交换的输入代币，最后一个地址代表期望获得的输出代币。在路径中的每个相邻地址之间，必须存在一个交易对合约。
   
     例如，假设你有代币 A，你想用它换取代币 C。如果市场上存在一个直接的交易对 A-C，那么你可以使用一个简单的路径：`[A, C]`。然而，如果不存在直接的 A-C 交易对，但存在与代币 B 的交易对，即 A-B 和 B-C，那么你可以通过代币 B 进行交换。在这种情况下，`path` 的格式将如下所示：`[A, B, C]`。
   
     这意味着交换过程将首先将代币 A 换成代币 B，然后将代币 B 换成代币 C。请注意，路径可以包含更多的代币，只要相邻的代币之间存在交易对即可。
   
     需要注意的是，当路径中涉及多个交易对时，交易可能会受到更大的价格滑点影响。这是因为每个交易对都会根据其储备量计算价格，而多个交易对的储备量可能会导致不同的价格。
   
   - `to`：接收输出代币的地址
   
   - `deadline`：交易的截止时间，用于确保交易在某个时间点之前完成
   
2. 调用 `UniswapV2Library.getAmountsIn` 函数，根据期望的输出代币数量和交换路径计算每个交易对中的输入金额。结果以数组形式返回。

3. 检查计算出的最初输入金额是否小于或等于用户指定的最大输入金额。如果不满足条件，则抛出异常。

4. 使用 `TransferHelper.safeTransferFrom` 函数将计算出的输入代币从用户地址转移到交易对合约地址。路径中的第一个代币和第二个代币构成交易对。

5. 调用内部函数 `_swap`，执行代币交换。根据传入的路径参数，`_swap` 函数可能完成单次或多次交换。

因此，`swapTokensForExactTokens`函数允许用户在 Uniswap V2 交易对中完成一个不精确输入、精确输出的代币兑换。用户需要指定期望的精确输出代币数量、允许的最大输入代币数量、代币交换路径和接收输出代币的地址。在执行交换时，函数会确保用户提供的输入代币数量不超过他们指定的最大输入代币数量。

