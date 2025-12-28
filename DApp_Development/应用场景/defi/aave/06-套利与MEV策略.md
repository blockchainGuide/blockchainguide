# Aave V3 套利与 MEV 策略

## 1. 概述

Aave V3 生态中存在多种套利机会：

| 策略类型 | 资金需求 | 技术难度 | 风险等级 |
|---------|---------|---------|---------|
| 清算套利 | 中等 | 中等 | 中等 |
| 利率套利 | 高 | 低 | 低 |
| 闪电贷套利 | 无 | 高 | 低 |
| 跨协议套利 | 高 | 中等 | 中等 |
| MEV 策略 | 高 | 高 | 高 |

---

## 2. 清算套利

### 2.1 清算机制回顾

```
清算触发条件: 健康因子 (HF) < 1

清算人获利 = 获得抵押品 - 支付债务
           = 债务 × (1 + 清算奖励) / 抵押品价格 - 债务

典型清算奖励: 5-10%
```

### 2.2 清算监控策略

```solidity
contract LiquidationMonitor {
    IPool public pool;
    IPoolDataProvider public dataProvider;

    struct UserPosition {
        address user;
        uint256 healthFactor;
        uint256 totalDebt;
        address[] collateralAssets;
        address[] debtAssets;
    }

    // 监控用户健康因子
    function checkLiquidation(address user) external view returns (
        bool canLiquidate,
        uint256 healthFactor,
        uint256 maxProfit
    ) {
        (
            uint256 totalCollateralBase,
            uint256 totalDebtBase,
            uint256 availableBorrowsBase,
            uint256 currentLiquidationThreshold,
            uint256 ltv,
            uint256 hf
        ) = pool.getUserAccountData(user);

        healthFactor = hf;
        canLiquidate = hf < 1e18;

        if (canLiquidate) {
            // 估算最大利润
            // 简化计算：假设 5% 清算奖励
            maxProfit = totalDebtBase * 5 / 100;
        }

        return (canLiquidate, healthFactor, maxProfit);
    }

    // 批量扫描用户
    function scanUsers(address[] calldata users) external view returns (
        address[] memory liquidatableUsers,
        uint256[] memory profits
    ) {
        uint256 count = 0;

        // 第一遍：计数
        for (uint256 i = 0; i < users.length; i++) {
            (bool canLiquidate, , ) = this.checkLiquidation(users[i]);
            if (canLiquidate) count++;
        }

        // 第二遍：填充
        liquidatableUsers = new address[](count);
        profits = new uint256[](count);
        uint256 idx = 0;

        for (uint256 i = 0; i < users.length; i++) {
            (bool canLiquidate, , uint256 profit) = this.checkLiquidation(users[i]);
            if (canLiquidate) {
                liquidatableUsers[idx] = users[i];
                profits[idx] = profit;
                idx++;
            }
        }
    }
}
```

### 2.3 闪电贷清算

```solidity
contract FlashLiquidator is IFlashLoanSimpleReceiver {
    IPool public immutable POOL;
    ISwapRouter public immutable SWAP_ROUTER;

    constructor(address pool, address router) {
        POOL = IPool(pool);
        SWAP_ROUTER = ISwapRouter(router);
    }

    function executeLiquidation(
        address collateralAsset,
        address debtAsset,
        address user,
        uint256 debtToCover,
        bool receiveAToken
    ) external {
        // 发起闪电贷
        POOL.flashLoanSimple(
            address(this),
            debtAsset,
            debtToCover,
            abi.encode(collateralAsset, user, receiveAToken),
            0
        );
    }

    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        require(msg.sender == address(POOL), "Invalid caller");
        require(initiator == address(this), "Invalid initiator");

        (address collateralAsset, address user, bool receiveAToken) =
            abi.decode(params, (address, address, bool));

        // 1. 批准池子使用债务资产
        IERC20(asset).approve(address(POOL), amount);

        // 2. 执行清算
        POOL.liquidationCall(
            collateralAsset,
            asset,
            user,
            amount,
            receiveAToken
        );

        // 3. 获取收到的抵押品数量
        uint256 collateralReceived;
        if (receiveAToken) {
            collateralReceived = IERC20(
                POOL.getReserveData(collateralAsset).aTokenAddress
            ).balanceOf(address(this));
            // 需要先取出 aToken
            POOL.withdraw(collateralAsset, collateralReceived, address(this));
            collateralReceived = IERC20(collateralAsset).balanceOf(address(this));
        } else {
            collateralReceived = IERC20(collateralAsset).balanceOf(address(this));
        }

        // 4. 将抵押品换成债务资产
        if (collateralAsset != asset) {
            IERC20(collateralAsset).approve(address(SWAP_ROUTER), collateralReceived);

            SWAP_ROUTER.exactInputSingle(ISwapRouter.ExactInputSingleParams({
                tokenIn: collateralAsset,
                tokenOut: asset,
                fee: 3000, // 0.3% 费率池
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: collateralReceived,
                amountOutMinimum: amount + premium, // 至少要能还贷
                sqrtPriceLimitX96: 0
            }));
        }

        // 5. 批准归还闪电贷
        uint256 amountOwed = amount + premium;
        IERC20(asset).approve(address(POOL), amountOwed);

        // 6. 转移利润给发起者
        uint256 profit = IERC20(asset).balanceOf(address(this)) - amountOwed;
        if (profit > 0) {
            IERC20(asset).transfer(tx.origin, profit);
        }

        return true;
    }
}
```

### 2.4 清算利润计算

```
示例场景：
- 用户债务: 10,000 USDC
- 抵押品: ETH @ $1,800
- 清算奖励: 5%
- 闪电贷费用: 0.09%
- DEX 滑点: 0.3%

计算：
1. 清算金额 = 10,000 × 50% = 5,000 USDC (假设 HF > 0.95)
2. 获得 ETH = 5,000 × 1.05 / 1,800 = 2.917 ETH
3. 卖出 ETH 获得 = 2.917 × 1,800 × (1 - 0.003) = 5,234 USDC
4. 闪电贷费用 = 5,000 × 0.0009 = 4.5 USDC
5. 净利润 = 5,234 - 5,000 - 4.5 = 229.5 USDC (约 4.6%)
```

---

## 3. 利率套利

### 3.1 固定/浮动利率套利

```solidity
contract RateArbitrage {
    IPool public pool;

    struct ArbitragePosition {
        address asset;
        uint256 borrowAmount;
        uint256 fixedRate;
        uint256 entryTime;
        bool isActive;
    }

    mapping(address => ArbitragePosition) public positions;

    // 当固定利率 < 预期浮动利率时开仓
    function openArbitrage(
        address asset,
        uint256 amount,
        uint256 expectedVariableRate
    ) external {
        DataTypes.ReserveData memory reserve = pool.getReserveData(asset);

        // 检查固定利率是否有利
        require(
            reserve.currentStableBorrowRate < expectedVariableRate,
            "Not profitable"
        );

        // 以固定利率借款
        pool.borrow(asset, amount, 1, 0, msg.sender); // 1 = STABLE

        positions[msg.sender] = ArbitragePosition({
            asset: asset,
            borrowAmount: amount,
            fixedRate: reserve.currentStableBorrowRate,
            entryTime: block.timestamp,
            isActive: true
        });
    }

    // 检查套利利润
    function checkProfit(address user) external view returns (
        int256 profit,
        uint256 duration
    ) {
        ArbitragePosition memory pos = positions[user];
        require(pos.isActive, "No position");

        DataTypes.ReserveData memory reserve = pool.getReserveData(pos.asset);

        // 计算利率差
        int256 rateDiff = int256(reserve.currentVariableBorrowRate) -
            int256(pos.fixedRate);

        duration = block.timestamp - pos.entryTime;

        // 简化计算：利润 = 借款金额 × 利率差 × 时间
        profit = (int256(pos.borrowAmount) * rateDiff * int256(duration)) /
            int256(365 days * 1e27);
    }
}
```

### 3.2 跨协议利率套利

```solidity
contract CrossProtocolArbitrage {
    IPool public aavePool;
    ICToken public compoundMarket;  // Compound cToken

    // 在 Compound 借入，在 Aave 存入
    function aaveSupplyCompoundBorrow(
        address asset,
        uint256 amount
    ) external {
        // 检查利率差是否有利可图
        uint256 aaveSupplyRate = _getAaveSupplyRate(asset);
        uint256 compoundBorrowRate = compoundMarket.borrowRatePerBlock() * 2102400; // 年化

        require(aaveSupplyRate > compoundBorrowRate, "Not profitable");

        // 从 Compound 借款
        compoundMarket.borrow(amount);

        // 存入 Aave
        IERC20(asset).approve(address(aavePool), amount);
        aavePool.supply(asset, amount, msg.sender, 0);
    }

    // 反向操作：在 Aave 借入，在 Compound 存入
    function compoundSupplyAaveBorrow(
        address asset,
        uint256 amount
    ) external {
        uint256 compoundSupplyRate = compoundMarket.supplyRatePerBlock() * 2102400;
        DataTypes.ReserveData memory reserve = aavePool.getReserveData(asset);
        uint256 aaveBorrowRate = reserve.currentVariableBorrowRate;

        require(compoundSupplyRate > aaveBorrowRate, "Not profitable");

        // 从 Aave 借款
        aavePool.borrow(asset, amount, 2, 0, msg.sender);

        // 存入 Compound
        IERC20(asset).approve(address(compoundMarket), amount);
        compoundMarket.mint(amount);
    }

    function _getAaveSupplyRate(address asset) internal view returns (uint256) {
        DataTypes.ReserveData memory reserve = aavePool.getReserveData(asset);
        return reserve.currentLiquidityRate;
    }
}
```

---

## 4. 闪电贷套利

### 4.1 DEX 价格套利

```solidity
contract FlashArbDex is IFlashLoanSimpleReceiver {
    IPool public pool;
    ISwapRouter public uniswap;
    ISwapRouter public sushiswap;

    function executeArbitrage(
        address tokenA,
        address tokenB,
        uint256 amount,
        bool buyOnUniswap
    ) external {
        pool.flashLoanSimple(
            address(this),
            tokenA,
            amount,
            abi.encode(tokenB, buyOnUniswap),
            0
        );
    }

    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        (address tokenB, bool buyOnUniswap) = abi.decode(params, (address, bool));

        uint256 amountOut;

        if (buyOnUniswap) {
            // Uniswap 买 → Sushiswap 卖
            IERC20(asset).approve(address(uniswap), amount);
            amountOut = uniswap.exactInputSingle(ISwapRouter.ExactInputSingleParams({
                tokenIn: asset,
                tokenOut: tokenB,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            }));

            IERC20(tokenB).approve(address(sushiswap), amountOut);
            amountOut = sushiswap.exactInputSingle(ISwapRouter.ExactInputSingleParams({
                tokenIn: tokenB,
                tokenOut: asset,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amountOut,
                amountOutMinimum: amount + premium,
                sqrtPriceLimitX96: 0
            }));
        } else {
            // Sushiswap 买 → Uniswap 卖
            // ... 类似逻辑
        }

        // 归还闪电贷
        IERC20(asset).approve(address(pool), amount + premium);

        return true;
    }
}
```

### 4.2 三角套利

```solidity
contract TriangularArbitrage is IFlashLoanSimpleReceiver {
    IPool public pool;
    ISwapRouter public router;

    function executeTriangularArbitrage(
        address tokenA,
        address tokenB,
        address tokenC,
        uint256 amount
    ) external {
        pool.flashLoanSimple(
            address(this),
            tokenA,
            amount,
            abi.encode(tokenB, tokenC),
            0
        );
    }

    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        (address tokenB, address tokenC) = abi.decode(params, (address, address));

        // A → B
        IERC20(asset).approve(address(router), amount);
        uint256 amountB = router.exactInputSingle(ISwapRouter.ExactInputSingleParams({
            tokenIn: asset,
            tokenOut: tokenB,
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amount,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        }));

        // B → C
        IERC20(tokenB).approve(address(router), amountB);
        uint256 amountC = router.exactInputSingle(ISwapRouter.ExactInputSingleParams({
            tokenIn: tokenB,
            tokenOut: tokenC,
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amountB,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        }));

        // C → A
        IERC20(tokenC).approve(address(router), amountC);
        uint256 finalAmountA = router.exactInputSingle(ISwapRouter.ExactInputSingleParams({
            tokenIn: tokenC,
            tokenOut: asset,
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amountC,
            amountOutMinimum: amount + premium,
            sqrtPriceLimitX96: 0
        }));

        // 确保盈利
        require(finalAmountA >= amount + premium, "Not profitable");

        // 归还
        IERC20(asset).approve(address(pool), amount + premium);

        return true;
    }
}
```

---

## 5. MEV 策略

### 5.1 清算 MEV

```solidity
contract LiquidationMEV {
    IPool public pool;

    // 使用 Flashbots 提交清算交易
    function submitLiquidation(
        address collateralAsset,
        address debtAsset,
        address user,
        uint256 debtToCover,
        uint256 minerTip
    ) external payable {
        // 执行清算
        pool.liquidationCall(
            collateralAsset,
            debtAsset,
            user,
            debtToCover,
            false
        );

        // 支付矿工小费
        block.coinbase.transfer(minerTip);
    }

    // 计算最优 Gas 价格
    function calculateOptimalGas(
        uint256 expectedProfit,
        uint256 gasUsed
    ) external pure returns (uint256 maxGasPrice) {
        // 留下 50% 利润作为安全边际
        maxGasPrice = (expectedProfit * 50 / 100) / gasUsed;
    }
}
```

### 5.2 Just-In-Time 流动性

```solidity
contract JITLiquidity {
    // 在大额交易前提供流动性，交易后撤回
    // 注意：这是高级 MEV 策略，需要与区块构建者合作

    function executeJIT(
        address pool,
        uint256 amount0,
        uint256 amount1,
        int24 tickLower,
        int24 tickUpper
    ) external {
        // 1. 在目标交易前添加流动性
        // 2. 让大额交易执行
        // 3. 在同一区块内移除流动性
        // 4. 收取交易费用

        // 这需要与 MEV 基础设施（如 Flashbots）配合
    }
}
```

### 5.3 后跑清算

```solidity
contract BackrunLiquidation {
    IPool public pool;

    // 监听 mempool 中的价格更新交易
    // 如果价格更新会导致清算，则后跑执行清算
    function backrunPriceUpdate(
        address user,
        address collateralAsset,
        address debtAsset,
        uint256 expectedDebtToCover
    ) external {
        // 检查用户是否可清算
        (, , , , , uint256 healthFactor) = pool.getUserAccountData(user);

        require(healthFactor < 1e18, "Not liquidatable");

        // 执行清算
        pool.liquidationCall(
            collateralAsset,
            debtAsset,
            user,
            expectedDebtToCover,
            false
        );
    }
}
```

---

## 6. 风险管理

### 6.1 清算风险

```solidity
contract RiskManager {
    // 滑点保护
    uint256 public constant MAX_SLIPPAGE = 300; // 3%

    // Gas 风险控制
    uint256 public constant MAX_GAS_PRICE = 500 gwei;

    // 最小利润阈值
    uint256 public constant MIN_PROFIT_BPS = 50; // 0.5%

    function validateLiquidation(
        uint256 debtToCover,
        uint256 expectedCollateral,
        uint256 currentGasPrice
    ) external view returns (bool isValid, string memory reason) {
        // 检查 Gas 价格
        if (currentGasPrice > MAX_GAS_PRICE) {
            return (false, "Gas too high");
        }

        // 估算利润
        uint256 expectedProfit = expectedCollateral * 105 / 100 - debtToCover;
        uint256 minProfit = debtToCover * MIN_PROFIT_BPS / 10000;

        if (expectedProfit < minProfit) {
            return (false, "Profit too low");
        }

        return (true, "");
    }
}
```

### 6.2 滑点保护

```solidity
contract SlippageProtection {
    // 计算最小输出（含滑点保护）
    function getMinOutput(
        uint256 amountIn,
        uint256 expectedPrice,
        uint256 maxSlippageBps
    ) external pure returns (uint256) {
        uint256 expectedOutput = amountIn * expectedPrice / 1e18;
        return expectedOutput * (10000 - maxSlippageBps) / 10000;
    }

    // 动态滑点（根据交易规模调整）
    function getDynamicSlippage(
        uint256 tradeSize,
        uint256 poolLiquidity
    ) external pure returns (uint256 slippageBps) {
        // 交易规模 / 池子流动性
        uint256 impact = tradeSize * 10000 / poolLiquidity;

        // 基础滑点 + 影响因子
        slippageBps = 30 + impact * 2; // 0.3% 基础 + 动态

        // 上限 5%
        if (slippageBps > 500) {
            slippageBps = 500;
        }
    }
}
```

### 6.3 竞争风险

```
清算竞争场景：
┌─────────────────────────────────────────────┐
│ 区块 N                                       │
│ ├─ Tx1: Bot A 清算 (Gas: 100 gwei)          │
│ ├─ Tx2: Bot B 清算 (Gas: 150 gwei) ← 胜出   │
│ └─ Tx3: Bot C 清算 (Gas: 80 gwei)           │
└─────────────────────────────────────────────┘

应对策略：
1. 使用 Flashbots 避免 Gas 竞争
2. 私有 mempool 交易
3. 与矿工/验证者合作
```

---

## 7. 监控工具

### 7.1 链下监控脚本

```javascript
// Node.js 监控示例
const { ethers } = require("ethers");

class AaveMonitor {
    constructor(provider, poolAddress, dataProviderAddress) {
        this.provider = provider;
        this.pool = new ethers.Contract(poolAddress, POOL_ABI, provider);
        this.dataProvider = new ethers.Contract(
            dataProviderAddress,
            DATA_PROVIDER_ABI,
            provider
        );
    }

    async scanForLiquidations(users) {
        const liquidatable = [];

        for (const user of users) {
            const data = await this.pool.getUserAccountData(user);
            const healthFactor = data.healthFactor;

            if (healthFactor.lt(ethers.utils.parseEther("1"))) {
                liquidatable.push({
                    user,
                    healthFactor: ethers.utils.formatEther(healthFactor),
                    totalDebt: ethers.utils.formatEther(data.totalDebtBase),
                    totalCollateral: ethers.utils.formatEther(data.totalCollateralBase)
                });
            }
        }

        return liquidatable;
    }

    async calculateLiquidationProfit(user, collateralAsset, debtAsset, debtToCover) {
        // 获取储备数据
        const collateralData = await this.dataProvider.getReserveData(collateralAsset);
        const debtData = await this.dataProvider.getReserveData(debtAsset);

        // 获取用户数据
        const userData = await this.pool.getUserAccountData(user);

        // 计算预期利润
        const liquidationBonus = collateralData.liquidationBonus; // 如 10500 = 105%
        const collateralReceived = debtToCover
            .mul(liquidationBonus)
            .div(10000);

        // 扣除各种费用后的净利润
        const flashLoanFee = debtToCover.mul(9).div(10000); // 0.09%
        const estimatedSlippage = collateralReceived.mul(30).div(10000); // 0.3%

        const netProfit = collateralReceived
            .sub(debtToCover)
            .sub(flashLoanFee)
            .sub(estimatedSlippage);

        return {
            grossProfit: collateralReceived.sub(debtToCover),
            netProfit,
            flashLoanFee,
            estimatedSlippage
        };
    }

    // 监听健康因子变化事件
    async monitorHealthFactors(callback) {
        // 监听借贷事件
        this.pool.on("Borrow", async (reserve, user, onBehalfOf, amount, ...args) => {
            const data = await this.pool.getUserAccountData(onBehalfOf);
            if (data.healthFactor.lt(ethers.utils.parseEther("1.1"))) {
                callback({
                    type: "warning",
                    user: onBehalfOf,
                    healthFactor: ethers.utils.formatEther(data.healthFactor)
                });
            }
        });

        // 监听清算事件
        this.pool.on("LiquidationCall", (...args) => {
            callback({
                type: "liquidation",
                args
            });
        });
    }
}
```

### 7.2 利率套利机会扫描

```javascript
async function scanRateArbitrage(aavePool, compoundMarkets) {
    const opportunities = [];

    for (const asset of SUPPORTED_ASSETS) {
        // Aave 利率
        const aaveData = await aavePool.getReserveData(asset);
        const aaveSupplyRate = aaveData.currentLiquidityRate;
        const aaveBorrowRate = aaveData.currentVariableBorrowRate;

        // Compound 利率
        const compoundMarket = compoundMarkets[asset];
        const compoundSupplyRate = await compoundMarket.supplyRatePerBlock();
        const compoundBorrowRate = await compoundMarket.borrowRatePerBlock();

        // 年化
        const BLOCKS_PER_YEAR = 2102400;
        const compoundSupplyAPY = compoundSupplyRate.mul(BLOCKS_PER_YEAR);
        const compoundBorrowAPY = compoundBorrowRate.mul(BLOCKS_PER_YEAR);

        // 检查套利机会
        // 策略1: Aave 存款 > Compound 借款
        if (aaveSupplyRate.gt(compoundBorrowAPY)) {
            opportunities.push({
                strategy: "Compound借 → Aave存",
                asset,
                spread: aaveSupplyRate.sub(compoundBorrowAPY),
                annualizedReturn: formatRate(aaveSupplyRate.sub(compoundBorrowAPY))
            });
        }

        // 策略2: Compound 存款 > Aave 借款
        if (compoundSupplyAPY.gt(aaveBorrowRate)) {
            opportunities.push({
                strategy: "Aave借 → Compound存",
                asset,
                spread: compoundSupplyAPY.sub(aaveBorrowRate),
                annualizedReturn: formatRate(compoundSupplyAPY.sub(aaveBorrowRate))
            });
        }
    }

    return opportunities.sort((a, b) => b.spread.sub(a.spread).toNumber());
}
```

---

## 8. 小结

| 策略 | 优势 | 风险 | 推荐度 |
|------|------|------|--------|
| **清算套利** | 利润确定、技术成熟 | 竞争激烈、Gas 成本 | ⭐⭐⭐⭐ |
| **利率套利** | 风险低、持续收益 | 利率变化、资金成本 | ⭐⭐⭐⭐⭐ |
| **闪电贷套利** | 无资金需求 | 技术复杂、利润薄 | ⭐⭐⭐ |
| **MEV 策略** | 利润高 | 技术门槛高、中心化风险 | ⭐⭐ |

**关键成功因素**：
1. **速度**：链上监控和执行速度
2. **Gas 优化**：降低交易成本
3. **风险控制**：滑点保护和利润阈值
4. **基础设施**：MEV 保护和私有交易

下一章将提供开发者集成指南和最佳实践。
