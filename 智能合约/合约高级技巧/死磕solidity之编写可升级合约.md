![image-20221028102022991](https://tva1.sinaimg.cn/large/008vxvgGgy1h7kse3u0fkj31g80t012r.jpg)



## 为什么要编写可升级合约

默认情况下，以太坊中的智能合约是不可变的。但是一旦项目方提前发现合约漏洞或者想升级功能，是需要合约可以变动的，因此一开始编写可升级的合约是重要的。因此我们使用OpenZeppelin Upgrades插件来实现我们可升级合约的目的。



## 合约升级原理

当您创建一个新的可升级合约实例时， 实际上部署了三个合约：

1. 您编写的合约，称为包含逻辑的实现合约
2. 代理的管理员合约
3. 代理合约，实际和用户交互的、

代理合约会把所有的调用委托给实现合约，代理持有状态数据，而实现合约提供代码。同时它还允许我们通过将代理委托给不同的实现合约来更改代码。从而实现了状态数据和实现的解耦。

![image-20221028113002214](https://tva1.sinaimg.cn/large/008vxvgGgy1h7kuengsfrj313s04yaa9.jpg)

代理需要解决的最直接的问题是代理如何公开逻辑合约的整个接口，而不需要对整个逻辑合约的接口进行一对一的映射。这将难以维护，容易出错，并且会使接口本身无法升级。因此，需要动态转发机制。下面的代码介绍了这种机制的基础知识：

```assembly
assembly {
  let ptr := mload(0x40)
  // (1) copy incoming call data
  calldatacopy(ptr, 0, calldatasize)
  // (2) forward call to logic contract
  let result := delegatecall(gas, _impl, ptr, calldatasize, 0, 0)
  let size := returndatasize
  // (3) retrieve return data
  returndatacopy(ptr, 0, size)
  // (4) forward return data back to caller
  switch result
  case 0 { revert(ptr, size) }
  default { return(ptr, size) }
}
```

此代码可以放在代理的回退函数中，并将对具有任何参数集的任何函数的任何调用转发到逻辑合约，而无需知道任何特别是逻辑合约的接口。本质上，（1） calldata复制到内存，（2）调用被转发到逻辑合约，（3）从调用到逻辑合约的返回数据被检索，以及（4）返回的数据被转发回来给来电者。

需要注意的一件非常重要的事情是，代码使用了 EVM 的 操作码，该操作码在调用者状态的上下文中执行被调用者的代码。也就是
说，逻辑合约控制着代理的状态，逻辑合约的状态是没有意义的。因此，代理不仅在逻辑合约之间转发交易，而且还代表交易对的状态。状态在代理中，逻辑在代理指向的特定实现中，详细的说明请参考官方文档。

## 透明代理



## UUPS

https://mshk.top/2022/06/solidity-hardhot-erc721-erc1822-erc1976-openzeppelin/

https://learnblockchain.cn/article/4534

https://mirror.xyz/xyyme.eth/kM9ld2u0D1BpHAfXTiaSPGPtDnOd6vrxJ5_tW4wZVBk

## 参考

> https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat
>
> https://blog.csdn.net/Tesla_Zhou/article/details/111743243
>
> https://learnblockchain.cn/article/4282
>
> https://docs.openzeppelin.com/upgrades-plugins/1.x/api-hardhat-upgrades
>
> https://xiaozhuanlan.com/topic/1762043598 大概的对所有的代理模式进行一个总结
>
> https://blog.gnosis.pm/solidity-delegateproxy-contracts-e09957d0f201 委托代理的深入理解
>
> https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/