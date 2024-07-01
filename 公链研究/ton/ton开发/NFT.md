## Ton类型

- 艺术品
- 收藏品
- 会员资格
  - https://ton.diamonds/collection/the-gateway/yyx-ticket-19
- TON DNS域名
  - https://ton.diamonds/zh/collection/ton-dns-domains?tab=items
- telegram username 
  - https://getgems.io/collection/EQCA14o1-VWhS2efqoh_9M1b_A9DtKTuoqfmkn83AbJzwnPi
- 匿名号码
  - 可创建于sim卡无关的telegram账号
- 代金券
  - no tcoin玩法 ： https://getgems.io/collection/EQDmkj65Ab_m0aZaW8IpKw4kYqIgITw_HRstYEkVQ6NIYCyW
  - 可转成真实代币



https://fragment.com/about

https://getgems.io/

https://ton.diamonds/

https://beta.disintar.io/collection/UQAv9X13hopdcDCO3azadyHy3mqIjzPgr0_fdrUdXjtpZmdK

## 基本实现

你要创建新的NFT项目，向集合合约发送消息，他就会根据你提供的数据创建一个NFT合约



## TON NFT 标准

- 每个NFT 物品和NFT 集合都是单独的智能合约,你发布了包含10000个项目的集合，则会部署100001个合约



## NFT元数据

每个NFT物品和NFT集合本身都有自己的元数据，元数据可以是链下（链接）或者链上（存储于智能合约中）

NFT物品元数据：

```json
{
   "name": "TON Smart Challenge #2 Winners Trophy",
   "description": "TON Smart Challenge #2 Winners Trophy 1 place out of 181",
   "image": "https://ton.org/_next/static/media/duck.d936efd9.png",
   "content_url": "https://ton.org/_next/static/media/dimond_1_VP9.29bcaf8e.webm",
   "attributes": []
}
```

NFT集合元数据：

```json
{
   "image": "https://ton.org/_next/static/media/smart-challenge1.7210ca54.png",
   "name": "TON Smart Challenge #2",
   "description": "TON Smart Challenge #2 Winners Trophy",
   "social_links": []
}
```



## 缺点

> 无法理解

由于 TON 是异步区块链，因此无法获取链上 NFT 的当前所有者。当包含有关 NFT 所有者的信息的消息被传递时，此信息可能会变得无关紧要，因此我们无法保证当前所有者没有发生变化。 



## 为什么他不是tokenId->owner_address 的单一合约

- 基于异步，他无法知道谁的消息会先到达，在复杂的交互链上（比如钱包到NFT合约再到拍卖合约），运行时无法确认字典大小（存储消耗gas）， 即无法预测消耗的gas,那么久可能出现前面的几步操作成功，到了拍卖失败了。 而如果是采用单个合约的话，合约的状态是不会随着执行改变的
- 无法扩展，TON在负载下会分为分片链，如果交易都是引用同一个合约，大大限制处理速度。 TON提供了分片智能合约，未实施



## 不存在approve

在同步区块链上，比如以太坊，智能合约可以一次性检查代币的所有权、批准和转账权限，确保所有条件都满足后才执行转账操作。然而，在TON这样的异步区块链上，由于消息处理的不确定性，这种方法不再适用。在TON上，为了确保操作的正确性，需要发送多条消息来执行相同的操作，并在每条消息的响应中检查是否出现错误，如果出现错误，则需要执行回滚操作。

这种方法不仅复杂，而且效率低下，因为它增加了网络交互的次数和潜在的错误处理成本。



## 存在的问题

- 还没有标准方法来安全转移， 合约执行失败，会恢复所有权的转移



