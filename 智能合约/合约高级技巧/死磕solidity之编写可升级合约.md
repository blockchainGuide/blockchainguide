![image-20221028102022991](https://tva1.sinaimg.cn/large/008vxvgGgy1h7kse3u0fkj31g80t012r.jpg)



## 为什么要编写可升级合约

默认情况下，以太坊中的智能合约是不可变的。但是一旦项目方提前发现合约漏洞或者想升级功能，是需要合约可以变动的，因此一开始编写可升级的合约是重要的。因此我们需要使用可升级的合约来增强可维护性。

## 升级合约概述

升级合约通常是采用代理模式来实现，这种模式的工作原理存在两个合约，一个是代理合约，一个是实现合约，代理合约负责管理合约状态数据，而实现合约只是负责执行合约逻辑，不存储任何状态数据。用户通过调用代理合约，代理合约对实现合约进行`delegate call`从而达到升级的目的。

![image-20221105184801563](https://tva1.sinaimg.cn/large/008vxvgGgy1h7ububbfr2j30av02xjrb.jpg)

目前主要有3种方式来替换/升级实现合约：

- Diamond Implementation
- Transparent Proxy Implementation
- UUPS Implementation

目前通用的是透明代理实现和UUPS实现，目的都是为了将实现合约的地址换成新的（升级后的合约），透明代理的方式是把更新实现合约函数`updatate to address`放在代理合约里，而UUPS是把更新实现合约放在实现合约中。

## 透明代理

透明代理（[EIP1967](https://eips.ethereum.org/EIPS/eip-1967)）是一种简单的方法来分离代理合约和合约之间的责任。在这种情况下，`upgradeTo`函数是代理合约中的一部分，而实现合约可以通过在代理上调用`upgradeTo`来升级，从而改变未来函数调用的委托位置。

不过也有一些注意事项。代理合约和实现合约如果有一个**名称和参数相同的函数**，在透明代理合约中，这个问题由代理合约来处理，代理合约根据`msg.sender`全局变量来决定用户的调用是在代理合约本身还是在实现合约中执行。

所以如果`msg.sender`是代理的管理员，那么代理将不会委托调用，如果它不是管理员地址，代理将把调用委托给实现合约，即使它与代理的某个函数相匹配。

所以透明代理存在此问题：`owner`的地址必须存储在存储器中，而使用存储器是与智能合约互动的最低效和最昂贵的步骤之一，每次用户调用代理时，代理会检查用户是否是管理员，这给大多数发生的交易增加了不必要的气体成本。（**总而言之成本比较高**）

## UUPS

UUPS代理（[EIP1822](https://eips.ethereum.org/EIPS/eip-1822)）是另一种方法来分离代理合约和合约之间的责任。UUPS代理模式下，`upgradeTo`函数是实现合约的一部分，并且通过代理合约被用户使用`delegatecall`。

在UUPS中，不管是管理员还是用户，所有的调用都被发送到实现合约中。这样做的好处是，每次调用时，我们不必访问存储空间来检查开始调用的用户是否是管理员，这提高了效率和成本。另外，因为是实现合约，你可以根据你的需要定制功能，在每一个新的实现中加入诸如`Timelock`、`Access Control`等，这在透明代理中是做不到的。

UUPS代理存在的问题是：`upgradeTo`函数存在于实现合约中，会增加不少代码，容易被攻击，并且如果开发者忘记添加这个函数，合约将不能再升级了。

## 使用OpenZeppelin编写可升级智能合约

### 透明代理实战

1. 安装 `hardhat` 环境

   ```shell
   ## 安装升级包
   $ yarn add @openzeppelin/contracts @openzeppelin/contracts-upgradeable @openzeppelin/hardhat-upgrades 
   
   ## 配置文件
   import { HardhatUserConfig } from 'hardhat/config'
   import '@nomicfoundation/hardhat-toolbox'
   import '@openzeppelin/hardhat-upgrades'
   
   const config: HardhatUserConfig = {
     solidity: '0.8.17'
   }
   
   export default config
   ```

2. 编写可升级合约

   ```solidity
   // SPDX-License-Identifier: MIT
   pragma solidity ^0.8.9;
   
   import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
   
   contract OpenProxy is Initializable {
       uint public value;
   
       function initialize(uint _value) public initializer {
           value = _value;
       }
   
       function increaseValue() external {
           ++value;
       }
   }
   
   ```

3. 部署脚本

   ```typescript
   import { ethers, upgrades } from 'hardhat'
   
   // yarn hardhat run scripts/deploy_openProxy.ts --network localhost
   async function main() {
       const OpenProxy = await ethers.getContractFactory('OpenProxy')
   
       // 部署合约, 并调用初始化方法
       const myOpenProxy = await upgrades.deployProxy(OpenProxy, [10], {
           initializer: 'initialize'
       })
   
       // 代理合约地址
       const proxyAddress = myOpenProxy.address
       // 实现合约地址
       const implementationAddress = await upgrades.erc1967.getImplementationAddress(myOpenProxy.address)
       // proxyAdmin 合约地址
       const adminAddress = await upgrades.erc1967.getAdminAddress(myOpenProxy.address)
   
       console.log(`proxyAddress: ${proxyAddress}`)
       console.log(`implementationAddress: ${implementationAddress}`)
       console.log(`adminAddress: ${adminAddress}`)
   }
   
   main().catch((error) => {
       console.error(error)
       process.exitCode = 1
   })
   
   ```

4. 编译合约&启动本地节点&本地网络部署合约

   ```ruby
   $ yarn hardhat compile
   $ yarn hardhat node
   $ yarn hardhat run scripts/proxy/open_proxy/openProxy.ts --network localhost
   
   ## 部署完毕
   proxyAddress: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
   implementationAddress: 0x5FbDB2315678afecb367f032d93F642f64180aa3
   adminAddress: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
   
   ```

实际上部署的合约有三个：

- 代理合约
- 实现合约
- `ProxyAdmin` 合约

`ProxyAdmin` 合约是用来管理代理合约的，包括了升级合约，转移合约所有权。

升级合约的步骤就是

- 部署一个新的实现合约，
- 调用 `ProxyAdmin` 合约中升级相关的方法，设置新的实现合约地址。

1. 新的实现合约

   ```solidity
   // SPDX-License-Identifier: MIT
   pragma solidity ^0.8.9;
   
   import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
   
   contract OpenProxyV2 is Initializable {
       uint public value;
   
       function initialize(uint _value) public initializer {
           value = _value;
       }
   
       function increaseValue() external {
           --value;
       }
   }
   ```

2. 升级脚本

   ```typescript
   import { ethers } from "hardhat";
   import { upgrades } from "hardhat";
   
   const proxyAddress = '0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0'
   
   async function main() {
       console.log(proxyAddress, " original proxy address")
       const OpenProxyV2 = await ethers.getContractFactory("OpenProxyV2")
       console.log("upgrade to OpenProxyV2...")
       const myOpenProxyV2 = await upgrades.upgradeProxy(proxyAddress, OpenProxyV2)
       console.log(myOpenProxyV2.address, " OpenProxyV2 address(should be the same)")
   
       console.log(await upgrades.erc1967.getImplementationAddress(myOpenProxyV2.address), " getImplementationAddress")
       console.log(await upgrades.erc1967.getAdminAddress(myOpenProxyV2.address), " getAdminAddress")
   }
   ...
   ```

   执行合约升级脚本如下：

   ```ruby
   0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0  original proxy address
   upgrade to OpenProxyV2...
   0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0  OpenProxyV2 address(should be the same)
   0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9  getImplementationAddress
   0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512  getAdminAddress
   ```

   可以发现代理合约地址和admin合约的地址并没有改变，仅仅是实现合约的地址发生了变化

以上通过 `upgrades.deployProxy` 部署的合约，默认情况下是使用的透明代理模式。如果你要想使用UUPS代理模式，需要显示的指定。

-----------------------------

### UUPS实战

hardhat环境还是以上，只不过有两个地方需要更改:

1. 编写合约

   ```solidity
   // SPDX-License-Identifier: MIT
   pragma solidity ^0.8.0;
   
   import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
   import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
   import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
   
   contract LogicV1 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
       function initialize() public initializer {
           __Ownable_init();
           __UUPSUpgradeable_init();
       }
   
       /// @custom:oz-upgrades-unsafe-allow constructor
       constructor() initializer {}
   
       // 需要此方法来防止未经授权的升级，因为在 UUPS 模式中，升级是从实现合约完成的，而在透明代理模式中，升级是通过代理合约完成的
       function _authorizeUpgrade(address) internal override onlyOwner {}
   
       mapping(string => uint256) private logic;
   
       event logicSetted(string indexed _key, uint256 _value);
   
       function SetLogic(string memory _key, uint256 _value) external {
           logic[_key] = _value;
           emit logicSetted(_key, _value);
       }
   
       function GetLogic(string memory _key) public view returns (uint256) {
           return logic[_key];
       }
   }
   
   ```

2. 合约部署脚本

   ```typescript
   ## 只需要稍微变动一下
   // 部署合约, 并调用初始化方法
   const myLogicV1 = await upgrades.deployProxy(LogicV1, {
     initializer: 'initialize',
     kind: 'uups'
   })
   ```

   ```typescript
   Warning: A proxy admin was previously deployed on this network
   // 管理员合约实际不存在了，只有代理合约和实现合约
   proxyAddress: 0xa513E6E4b8f2a923D98304ec87F64353C4D5C853
   implementationAddress: 0x5FC8d32690cc91D4c39d9d3abcBD16989F875707
   adminAddress: 0x0000000000000000000000000000000000000000
   ```

   编译并部署 UUPS代理模式的合约时，实际只会部署两个合约

   - 代理合约
   - 实现合约

   此时的升级合约的步骤就是

   - 部署一个新的实现合约，
   - 调用 `ProxyAdmin` 合约中升级相关的方法，设置新的实现合约地址。

   ---------------------------------

```
**************************
*****wx: mindcarver*******
*****公众号:区块链技术栈*****
**************************
```



## 参考

> [文章源码](https://github.com/blockchainGuide)
>
> https://eips.ethereum.org/EIPS/eip-1822
>
> https://eips.ethereum.org/EIPS/eip-1967
>
> https://blog.openzeppelin.com/the-state-of-smart-contract-upgrades/
>
> https://blog.gnosis.pm/solidity-delegateproxy-contracts-e09957d0f201
>
> https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/
>
> https://docs.openzeppelin.com/upgrades-plugins/1.x/api-hardhat-upgrades
>
> https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat