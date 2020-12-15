> 死磕以太坊源码分析之Ethash共识算法
>
> 代码分支：https://github.com/ethereum/go-ethereum/tree/v1.9.9

## 引言

目前以太坊中有两个共识算法的实现：`clique`和`ethash`。而`ethash`是目前以太坊主网（`Homestead`版本）的`POW`共识算法。

## Ethash 设计原理

以太坊设计共识算法时，期望达到三个目的：

1. 抗`ASIC`性：为算法创建专用硬件的优势应尽可能小，让普通计算机用户也能使用CPU进行开采。
   - 通过内存限制来抵制（`ASIC`使用矿机内存昂贵）
   - 大量随机读取内存数据时计算速度就不仅仅受限于计算单元，更受限于内存的读出速度。
2. 轻客户端可验证性: 一个区块应能被轻客户端快速有效校验。
3. 矿工应该要求存储完整的区块链状态。

### 哈希数据源

`ethash`要计算哈希，需要先有一块数据集。这块数据集较大，初始大小大约有`1G`，每隔 3 万个区块就会更新一次，且每次更新都会比之前变大`8M`左右。计算哈希的数据源就是从这块数据集中来的；而决定使用数据集中的哪些数据进行哈希计算的，才是`header`的数据和`Nonce`字段。这部分是由`Dagger`算法实现的。

#### Dagger

`Dagger`算法是用来生成数据集`Dataset`的，核心的部分就是`Dataset`的生成方式和组织结构。

可以把`Dataset`想成多个`item`（**dataItem**）组成的数组，每个`item`是`64`字节的byte数组（一条哈希）。`dataset`的初始大小约为`1G`，每隔3万个区块（一个`epoch`区间）就会更新一次，且每次更新都会比之前变大`8M`左右。

`Dataset`的每个`item`是由一个缓存块（`cache`）生成的，缓存块也可以看做多个`item`（**cacheItem**）组成，缓存块占用的内存要比`dataset`小得多，它的初始大小约为`16M`。同`dataset`类似，每隔 3 万个区块就会更新一次，且每次更新都会比之前变大`128K`左右。

生成一条`dataItem`的程是：从缓存块中“随机”（这里的“随机”不是真的随机数，而是指事前不能确定，但每次计算得到的都是一样的值）选择一个`cacheItem`进行计算，得的结果参与下次计算，这个过程会循环 256 次。

缓存块是由`seed`生成的，而`seed`的值与块的高度有关。所以生成`dataset`的过程如下图所示：

![image-20201213144908721](https://tva1.sinaimg.cn/large/0081Kckwgy1glm8arq9t7j30x80eytji.jpg)

`Dagger`还有一个关键的地方，就是确定性。即同一个`epoch`内，每次计算出来的`seed`、缓存、`dataset`都是相同的。否则对于同一个区块，挖矿的人和验证的人使用不同的`dataset`，就没法进行验证了。

----

### 生成哈希值（待定）

#### Hashimoto

`Hashimoto`是使用区块`Header`的哈希和`Nonce`字段、利用`dataset`数据，生成一个最终的哈希值。

hashimoto算法先用区块的Header哈希和Nonce值生成一个初始哈希，然后通过简单的计算(generateIndexBy)得到需要获取的dataset数据中的item索引。取到item以后，将其聚合到当前的mix变量中。

-----



## 共识引擎核心函数

代码位于`consensus.go`

![image-20201214150532321](https://tva1.sinaimg.cn/large/0081Kckwgy1glnee6vhalj31py0lkgq4.jpg)

①：`Author`

```go
// 返回coinbase, coinbase是打包第一笔交易的矿工的地址
func (ethash *Ethash) Author(header *types.Header) (common.Address, error) {
	return header.Coinbase, nil
}
```

②：`VerifyHeader`

主要有两步检查，第一步检查**header是否已知**或者**是未知的祖先**，第二步是`ethash`的检查：

2.1 header.Extra 不能超过32字节

```go
if uint64(len(header.Extra)) > params.MaximumExtraDataSize {  // 不超过32字节
		return fmt.Errorf("extra-data too long: %d > %d", len(header.Extra), params.MaximumExtraDataSize)
	}
```

2.2 时间戳不能超过15秒，15秒以后的就被认定为未来的块

```go
if !uncle {
		if header.Time > uint64(time.Now().Add(allowedFutureBlockTime).Unix()) {
			return consensus.ErrFutureBlock
		}
	}
```

2.3 当前header的时间戳小于父块的

```go
if header.Time <= parent.Time { // 当前header的时间小于等于父块的
		return errZeroBlockTime
	}
```

2.4 根据时间戳和父块的难度来验证块的难度

```go
expected := ethash.CalcDifficulty(chain, header.Time, parent)
	if expected.Cmp(header.Difficulty) != 0 {
		return fmt.Errorf("invalid difficulty: have %v, want %v", header.Difficulty, expected)
	}
```







## 总结&参考

> https://mindcarver.cn
>
> https://github.com/blockchainGuide
>
> https://eth.wiki/concepts/ethash/design-rationale
>
> https://eth.wiki/concepts/ethash/dag