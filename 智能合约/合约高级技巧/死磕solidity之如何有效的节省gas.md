![image-20221028092606674](https://tva1.sinaimg.cn/large/008vxvgGgy1h7kqts12j2j31210u0k44.jpg)

## 为什么要强调优化gas的重要性

DAPP中收取的费用取决于功能逻辑的复杂程度，越复杂消耗的计算资源越多。并且需要用户承担一部分gas,所以solidity 的优化显得非常的重要。同时注重优化gas的合约开发人员写出来的合约代码更安全，质量更高。

## 1. 封装结构

以uint 为例，如果我们的程序中包含多个类似的变量，可以将其封装在一起，因为不管uint8 ,uint32 ,uint16,solidity都会为其保留256位。即使你使用uint8也不会节省gas.



## 2. 最小化读写链上数据

首先明确一点在读写 memory 变量比读写 storage 变量便宜。

```solidity
contract NotSaveGas {
uint public var1  = 70;
function f1() external view returns (uint) {
        uint sum = 0;
        for (uint i = 0; i < var1; i++) {
            sum += i;
        }
        return sum;
}

contract SaveGas {
function f2() external view returns (uint) {
        uint sum = 0;

        for (uint i = 0; i < var1; i++) {
            sum += i;
        }
        return sum;
	}
}
```

请一定要避免f1这种循环读写 storage 变量，这是比较消耗gas的方式。处理这种问题实际可以定义内存变量作为缓存，将数据写入，这样可以节省大量的gas.

## 3.打开 solidity 优化器

hardhat 配置：

```javascript
module.exports = {
  solidity: {
    version: "0.8.9",
    settings: {
      optimizer: {
        enabled: true,
        runs: 1000,
      },
    },
  },
}
```



## 4.尽可能减少链上数据

区块链上保存数据是非常昂贵的，所以需要尽可能将链上存储的信息减少，以此来节省大量的交易gas.

### 使用事件

事件是外部事物(例如用户界面)从区块链中获得通知的内置方式。当发出事件时，将通知该事件的监视者。更新合约变量时不会发生通知。事件以不同的方式存储，比使用合约存储便宜得多。合约不能直接访问日志。

### IPFS

如果你需要存储文件之类，可以使用IPFS保存文件，并将存储的ID保存在链上。

### 无状态合约

### Merkle Proofs

如果需要使用区块链来验证一些信息是否有效，可以使用 merkle 证明。Merkle 证明使用单一的数据块来证明更大的数据量的有效性。例如，如果有人想证明 "Tx4 "的有效性，他将需要提供 Tx4、Hash3、Hash12 和 Hash5678，然后你的合约将能够重新计算 Merkle 根（Hash12345678），并检查它是否与存储在区块链上的根相一致。你将不需要存储所有交易的哈希值。

![image-20221027142237776](https://tva1.sinaimg.cn/large/008vxvgGgy1h7jtsvns3ej31280r2wi0.jpg)

-------

## 5.注重变量顺序

Solidity 存储槽的长度为 32 字节，但并不是所有的数据类型都需要这么大的空间：bool, int8 ... int128, bytes1 ... bytes31 和地址需要的空间小于 32 字节。solidity 会尝试将这些变量打包到同一个存储槽中。

如果你接连定义了 2 个uint128，它们都会被打包到同一个存储槽中，因为它们各占 16 字节。然而，如果你定义了一个uint128，接着是一个unit256，然后是另一个int128，你将使用 3 个存储槽，因为在两个 int128 之间的 unit256 需要一个完整的存储槽。

```solidity
contract T{
	// 不好的方式
	uint128 v1;
	uint256 v2;
	uint128 v3;
	
	// 推荐方式
	uint128 v1;
	uint128 v3;
	uint256 v2;
}
```

## 6.首选数据类型

如果智能合约只需要一个状态变量，一个永远不会大于 255 的无符号整数。我们常规思想可能是想用uint8,会觉得节省gas,实际并不会。以太坊操作码被设计为使用 256 位的变量（EVM 堆栈的大小），而 uint8 只需要 8 位，EVM 会在剩余的位上填上 "0"，以便能够操作它。这个由 EVM 执行的填 "0" 操作将花费 Gas，因此为了节省交易 Gas，最好使用 uint256 而不是 uint8。

## 7.独立部署库

如果在智能合约中重复使用代码，最好是将所有的代码打包到一个库中，并通过import的方式指向它。

库包含：

- 嵌入式库：包含内部函数的库，这些库都是嵌入在合约中，和合约一同部署，所以会比较消耗gas
- 独立库：包含public和外部函数的库，这些库只会被部署一次，同时被所有导入它的所有合约使用，从而节省了大量的gas.

## 8.构造函数

常量和不可变的状态变量在合约被部署后不能被改变。区别在于，常量必须在编译时定义，而不可变量可以在构造函数中定义。总是尽量使用常量，以便使构造函数更便宜。

## 9.使合约尽可能的小

单个合约的限制是24KB，所以要想节省gas,就必须使实现的合约尽可能的小。

- revert和assert的提示信息要尽可能的短
- 修改器： 修改器（modifier）代码是内联的，这意味着它会被添加在所修改的函数的开头或结尾。在使用修改器时减少合约大小的一个技巧是编写一个实现修改器逻辑的函数，然后让修改器调用该函数。这样实现修改器的代码就不会被复制，只有函数调用会被复制。这种技术只在同一修改器被多次使用时有效。

```solidity
modifier TestModifier(uint256 value){
		JudgeLength(value);
		_;
}
function JudgeLength(uint256 value)internal{
	//logic
}
```

## 10.最小代理（ERC1167）

如果需要部署多个功能完全相同的合约，应该考虑使用 "最小代理"（在ERC 1167中定义）

最小的代理只是一个合约，它将把所有的调用委托给一个预先定义的*实现合约*。它有一个定义好的字节码，代表最小代理合约的编译代码，你只需要把你的实现合约地址插入其中，你就可以根据需要部署最小代理的多个副本。 参考[ERC 1167](https://learnblockchain.cn/tags/EIP1167) 相关文章，了解如何使用最小代理）。

由于最小的代理字节码非常小，部署它的成本也低到不能再低，因此节省一堆**部署 Gas**。

使用最小代理的注意事项，你应该牢记：最小代理的实现合约地址不能改变，这意味着你将不能升级他们的代码。

## 11.内存位置

以太坊存在4个内存位置，从最便宜到最贵的：calldata、stack、memory、storage。

- calldata：只适用于输入参数且参数是外部函数的引用数据类型（数组，字符串 ...）。Calldata 参数是只读的，如果你有一些需要传递给函数的引用类型，总是考虑使用 calldata，因为它是最便宜的。
- stack：只对方法中定义的值类型数据有效。
- memory：内存是易丢失的 RAM，在 EVM 终止运行的时候会被移除。可以用它来存储引用数据类型，它比storage变量更便宜。当向其他函数传递参数，或在你的函数中声明临时变量时，除非你严格需要使用storage变量，否则应该总是使用 memory变量。
- storage：是最昂贵的存储位置。存储数据在区块链上持久存在，请尽量减少链上数据存储。
- 本地存储变量：本地存储变量是方法的本地变量，它指向一个实际的状态变量（存储在区块链存储中）。与其在内存中复制/粘贴存储数组以便操作它们，然后将它们复制回存储，不如简单地使用本地存储变量，直接在存储上操作。
- 批处理：与其让用户用不同的值多次调用同一个函数（通过向区块链发送多个交易），不如让他们通过传递动态大小的数组，以便可以在一个单一的交易中批量执行相同的功能。这将能够节省一些交易基础开销成本。实际ERC1155有些思想就是如此



## 12.尽量减少链上操作

- 字符串：如果可以使用bytes,则尽量使用。如果仍然需要操作，则尽量放在智能合约外部操作。
- 返回值：对返回值无需额外转换，如果这个是可以通过链外数据来处理。
- 循环：**避免在长数组中循环**，这不仅会花费大量的 Gas，而且如果 Gas 成本增加到很高的程度（超过 BlockGas 限制），会使合约无法执行。使用映射来代替长数组，映射是一个哈希表，可以让你在一次操作中使用其键来访问任何值，而不是在数组中循环，直到找到你要找的键。

## 13.利用 view函数减少gas

当用户从外部调用一个view函数，是不需要支付一分 gas 的。

这是因为 view 函数不会真正改变区块链上的任何数据 - 它们只是读取。因此用 view 标记一个函数，意味着告诉 web3.js，运行这个函数只需要查询你的本地以太坊节点，而不需要在区块链上创建一个事务（事务需要运行在每个节点上，因此花费 gas）。

在所能只读的函数上标记上表示“只读”的“external view 声明，就能为你的玩家减少在 DApp 中 gas 用量。

注意：如果一个 view 函数在另一个函数的内部被调用，而调用函数与 view 函数的不属于同一个合约，也会产生调用成本。这是因为如果主调函数在以太坊创建了一个事务，它仍然需要逐个节点去验证。所以标记为 view 的函数只有在**外部调用**时才是免费的。

----

## 14.使用短路模式排序solidity操作

短路（short-circuiting）是一种使用或/与逻辑来排序不同成本操作的solidity合约 开发模式，它将低gas成本的操作放在前面，高gas成本的操作放在后面，这样如果前面的低成本操作可行，就可以跳过（短路）后面的高成本以太坊虚拟机操作了。

```cpp
// f(x) 是低gas成本的操作
// g(y) 是高gas成本的操作

// 按如下排序不同gas成本的操作
f(x) || g(y)
f(x) && g(y)
```

## 15.删除不必要的库

在开发Solidity智能合约时，我们引入的库通常只需要用到其中的部分功能，这意味着其中可能会包含大量对于你的智能合约而言其实是冗余的solidity代码。如果可以在你自己的合约里安全有效地实现所依赖的库功能，那么就能够达到优化solidity合约的gas利用的目的。

例如，在下面的solidity代码中，我们的以太坊合约只是用到了SafeMath库的`add`方法：

```solidity
import './SafeMath.sol' as SafeMath;

contract SafeAddition {
 function safeAdd(uint a, uint b) public pure returns(uint) {
 return SafeMath.add(a, b);
 }
}
```

通过参考SafeMath的这部分代码的实现，可以把对这个solidity库的依赖剔除掉：

```solidity
contract SafeAddition {
 function safeAdd(uint a, uint b) public pure returns(uint) {
 uint c = a + b;
 require(c >= a, "Addition overflow");
 return c;
 }
}
```

## 16.精确的声明函数的可见性

在Solidity合约开发中，显式声明函数的可见性不仅可以提高智能合约的安全性， 同时也有利于优化合约执行的gas成本。例如，通过显式地标记函数为外部函数（External），可以强制将函数参数的存储位置设置为`calldata`，这会节约每次函数执行时所需的以太坊gas成本。

> External 可见性比 public 消耗gas 少。

## 17.避免代码中死代码

死代码（Dead code）是指那些永远也不会执行的Solidity代码，例如那些执行条件永远也不可能满足的代码，就像下面的两个自相矛盾的条件判断里的Solidity代码块，消耗了以太坊gas资源但没有任何作用：

```solidity
function deadCode(uint x) public pure {
 if(x < 1 {
    if(x > 2) {
        return x;
    }
 }
}
```

## 18.避免使用常量结果的循环

如果一个循环计算的结果是无需编译执行Solidity代码就可以预测的，那么 就不要使用循环，这可以可观地节省gas。例如下面的以太坊合约代码就可以 直接设置num变量的值：

```solidity
function constantOutcome() public pure returns(uint) {
  uint num = 0;
  for(uint i = 0; i < 100; i++) {
    num += 1;
  }
  return num;
}
```

## 19.合并循环

有时候在Solidity智能合约中，你会发现两个循环的判断条件一致，那么在这种情况下就没有理由不合并它们。例如下面的以太坊合约代码：

```solidity
function loopFusion(uint x, uint y) public pure returns(uint) {
  for(uint i = 0; i < 100; i++) {
    x += 1;
  }
  for(uint i = 0; i < 100; i++) {
    y += 1;
  }
  return x + y;
}
```

## 20.去除循环中的比较运算

如果在循环的每个迭代中执行比较运算，但每次的比较结果都相同，则应将其从循环中删除。

```solidity
function unilateralOutcome(uint x) public returns(uint) {
  uint sum = 0;
  for(uint i = 0; i <= 100; i++) {
    if(x > 1) {
      sum += 1;
    }
  }
  return sum;
} 
```



```
********************
最终的定稿 ： https://github.com/blockchainGuide/blockchainguide
公众号 ： 区块链技术栈
********************
```



---------

## 参考

> https://medium.com/coinmonks/smart-contracts-gas-optimization-techniques-2bd07add0e86