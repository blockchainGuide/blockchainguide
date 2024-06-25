

## bitlayer桥的设计

【质押】

1. 用户质押方式有两种：

   - DLC

   - N of bitvm federation address  （确认这个方案，任意金额，速度更快，且没有CET限制），这里要参考下bitvm 的桥通道

2. 链下relayer  观察BITCOIN（事件）
3. Bitlayer l2层更新bitcoin的轻客户端
4. 然后在 l2桥合约上mint (实际应该有个btc-l2 地址，这是用户自己在L2的地址，私钥自己控制) 

【提现】

1. 用户提款，直接将在L2上的资产burn掉，然后relayer 监控L2的withdraw事件 ，然后要区分是DLC还是bitvm 的withdraw 
   - DLC withdraw 
   - BitVm withdraw

### 角色

Relayer :

- 监控比特币网络，将最新的比特币区块信息提交到l2轻客户端合约



？

ALice 的存款锁定交易达到7个之后，任何一方（？）向Layer2 桥合约提交铸币交易，同时必须附带SPV证明

L2桥合约：

桥验证



## 实施方案

- bitvm 联盟节点 的实现 （tendermint 节点或者mpc 签名方案）

- DLC的实现，需要预先设置CET

  ```
  // Alice和BitVM联盟的2 of 2多签名输出
  AlicePublicKey = "AlicePublicKey"
  BitVMPublicKey = "BitVMPublicKey"
  
  // 创建一个2 of 2多签名输出
  output1 = { value: 5, script: 2 <AlicePublicKey> <BitVMPublicKey> 2 CHECKMULTISIG }
  
  // CET1：1个月后Alice提取1 BTC
  CET1PublicKey = "CET1PublicKey"
  CET1AlicePrivateKey = "CET1AlicePrivateKey"
  
  // Alice在1个月后提交CET1AlicePrivateKey来解锁1 BTC
  CET1 = { value: 1, script: 1 <AlicePublicKey> CHECKSIGVERIFY 1 <CET1PublicKey> CHECKSIG }
  
  // CET2：3个月后Alice提取2 BTC
  CET2PublicKey = "CET2PublicKey"
  CET2AlicePrivateKey = "CET2AlicePrivateKey"
  
  // Alice在3个月后提交CET2AlicePrivateKey来解锁2 BTC
  CET2 = { value: 2, script: 1 <AlicePublicKey> CHECKSIGVERIFY 1 <CET2PublicKey> CHECKSIG }
  
  // Alice和BitVM联盟的交易
  transaction = {
    inputs: [ ... ],
    outputs: [ output1, CET1, CET2 ],
    locktime: 1 // 锁定时间1个月
  }
  
  ```

- L2层的合约已经有了，需要研究一下 

- 轻客户端实际也是一个合约，他是通过zkp来维护的（比特币的状态证明来更新维护比特币header），同时还提供验证比特币交易的合法性，通过将比特币上的lock交易的spv证明提交给合约来验证。验证成功，L2才会发行XBTC给用户

- Relayer 的实现 （无信任的机制）转发交易到L2智能合约，或者转发到bitvm 联盟节点



## bitlayer 合约

【BTR】

创建10亿个BTR，分配个不同的地址账户

【LockingContract】

按照预设的禀赋期和解锁期,逐步释放受益人锁仓的代币。在禀赋期内,受益人无法领取任何代币;在解锁期内,可根据时间的推移按比例领取。受益人可更改地址,也可领取当前已解锁的部分代币。合约的构造函数需要事先设置好所有参与者的锁仓计划。

【MultiSigWallet】

合约使用了OpenZeppelin的ECDSA(用于签名验证)和EnumerableSet(用于存储签名人和待执行交易)库。它通过事件来记录交易的生命周期。整个流程需要多个签名人的参与,从而实现了对资金的多重控制和审查,提高了安全性。应用场景包括多签名钱包、DAO等需要多方共同决策的场合。

【token factory】

这个合约充当了一个ERC20代币的工厂,允许管理员批量创建和铸造符合ERC20标准的代币,并使用了Create2确定性部署技术。可以用于需要快速发行大量ERC20代币的场景,如ICO、Airdrop等。

【vault】

总的来说,这个Vault合约提供了一个安全的存管库,由管理员控制对资产的发放,只有白名单地址才能接收发放的资产。它可以持有和发放原生代币(如ETH)和ERC20代币资产。常见的使用场景包括ICO、Airdrop等活动的资金存管,确保资金的安全管理和受控发放

【WBTC】

1. **存款(deposit)**:任何人都可以通过向合约转账ETH来存款,存入的ETH数量将被视为等值的WBTC数量,并记录在该用户的WBTC余额中。receive函数会调用deposit函数完成存款。
2. **取款(withdraw)**: 持有WBTC余额的用户可以提取等值的ETH。取款时,用户的WBTC余额会被相应减少

【params】

这个名为 `Params` 的合约定义了一个区块链系统中使用的各种常量参数和修饰符。它的主要作用是:

1. **定义系统常量**:
   - `COEFFICIENT`: 用于奖励计算时的扩大倍数。
   - `engineCaller`: 引擎调用者的硬编码地址。
   - `MaxValidators`: 系统中允许的最大验证者数量。
   - `MaxBackups`: 系统中允许的最大备用验证者数量。
   - `MaxStakes`: 每个验证者的最大总质押tokens。
   - `ThresholdStakes`: 验证者成为有效候选者所需的最小总质押。
   - `MinSelfStakes`: 用户注册验证者所需的最小自我质押。
   - `StakeUnit`: 质押的单位,这里是1个以太币。
   - `JailPeriod`: 验证者被监禁的区块数量(大约3天)。
   - `UnboundLockPeriod`: 验证者解绑质押时的延迟时间(28天)。
   - `PunishBase`: 惩罚计算的基础值。
   - `LazyPunishFactor`: 验证者未能按时出块时的惩罚系数。
   - `LazyPunishThreshold`: 对验证者进行惩罚的累计缺块数量。
   - `DecreaseRate`: 每个验证者在一个周期内允许的最大缺块数量。

这个合约似乎是一个较大系统的一部分,用于管理区块链网络中的验证者、质押和惩罚相关的参数。这些常量定义了验证者、质押和惩罚计算中的各种限制、阈值和参数。修饰符用于限制对某些函数的访问权限,基于调用者的地址或传入地址的有效性。



【staking】

这个合约名为 `Staking`，主要用于管理一个基于权益证明(PoS)共识机制的区块链网络中的验证者(Validator)。它具有以下主要功能:

1. **初始化和配置**
   - 初始化验证者参数,如最大验证者数量、最小质押要求等
   - 在创世块时初始化一批初始验证者

2. **验证者注册和管理**
   - 支持新验证者注册(可在无需许可或需要管理员许可的情况下注册)
   - 验证者可以增加或减少自身的质押金额
   - 用户可以为验证者delegation(委托)或从验证者取消delegation

3. **奖励分发**
   - 区块手续费将按比例分发给活跃验证者和备用验证者
   - 计算并更新每个验证者应得的奖励

4. **惩罚机制** 
   - 惩罚未能正常出块的验证者,按照错过的块数量扣减其质押
   - 采用渐进惩罚策略,错过一定数量块时扣除质押

5. **验证者集更新**
   - 链上组件可调用函数更新活跃验证者集和备用验证者集
   - 根据总质押量动态调整验证者排名

6. **查询功能**
   - 查询活跃/备用验证者列表
   - 查询任意地址在某验证者处的可提取金额(含质押和奖励)

该合约通过初始化、注册、奖惩等机制来维护和管理整个验证者集体,确保网络的正常运行和安全性。它实现了链上管理验证者所需的各种功能。



【va lidator】

1. **验证者状态与配置**：
   - 合约中包含一个验证者的地址（`validator`），该地址参与共识过程。
   - 验证者可设置佣金率（`commissionRate`），用于收取委托人收益的一部分作为管理费用。
   - 自持权益（`selfStake`）代表验证者自己投入的权益。
   - 总权益（`totalStake`）等于自持权益加上所有其他委托人的权益。
   - `acceptDelegation` 标志表明验证者是否接受委托。
   - `state` 表示验证者的当前状态（如就绪、退出等），并可以进行状态变更。
   - 合约还维护了其他相关参数，如退出锁定结束时间（`exitLockEnd`）、惩罚区块高度（`punishBlk`）、累计惩罚因子（`accPunishFactor`）等。
2. **委托管理**：
   - `Delegation` 结构体记录了每个委托人的详细信息，如是否存在委托关系、权益数量、已结算奖励、债务等。
   - `allDelegatorAddrs` 存储所有委托人的地址列表，便于遍历。
   - `delegators` 映射表以委托人地址为键，存储对应的 `Delegation` 结构体，管理各个委托人的权益和状态。
   - `PendingUnbound` 和 `UnboundRecord` 结构体及相关映射表用于跟踪委托人待解除绑定的权益及其锁定状态。
3. **权益操作**：
   - `addStake` 函数允许验证者增加自持权益，同时更新累计奖励和内部债务，并可能触发排名操作。
   - `subStake` 函数允许验证者减少自持权益，可选择是否将减少的权益标记为待解除绑定。函数还会调整总未提取权益。
   - `exitStaking` 函数使验证者退出staking状态，结算奖励，添加待解除绑定记录，并更新状态。
4. **奖励与费用处理**：
   - `receiveFee` 函数处理收到的费用，根据佣金率分配给验证者和委托人，并更新累计奖励率。
   - `validatorClaimAny` 函数允许验证者领取其应得的奖励（包括staking奖励、佣金和费用），同时处理待解除绑定的权益。奖励通过 `_recipient` 地址发送出去，并触发 `RewardsWithdrawn` 事件。
5. **访问控制与限制**：
   - 使用 `onlyOwner` 和 `onlyAdmin` 修饰符保护仅允许特定地址（如 Staking 合约地址或管理员地址）调用某些方法。
   - 使用 `onlyCanDoStaking` 修饰符确保在允许staking的状态下才能进行权益增减操作。
   - 使用 `onlyValidRate` 修饰符确保佣金率在有效范围内（0到100之间）。
6. **添加委托 (`addDelegation`)**
   - 确保验证者接受委托且总权益不超过最大值。
   - 如果委托人为新用户，将其添加至 `allDelegatorAddrs` 列表。
   - 调用 `handleDelegatorPunishment` 处理委托人可能的惩罚。
   - 更新委托人的债务、权益，并调用 `addTotalStake` 更新总权益及触发排名操作。
   - 返回排名操作结果。
7. **减少委托 (`subDelegation`)**
   - 调用 `handleDelegatorPunishment` 处理委托人可能的惩罚。
   - 转调 `innerSubDelegation` 实现减少委托的具体逻辑。
8. **退出委托 (`exitDelegation`)**
   - 确保委托人有权益。
   - 调用 `handleDelegatorPunishment` 处理惩罚。
   - 调用 `innerSubDelegation` 减少全部权益，并设置 `_isUnbound` 为 `true`，将全部权益标记为待解除绑定。
   - 返回排名操作结果及减少的权益数量。
9. **内部减少委托 (`innerSubDelegation`)**
   - 更新委托人的已结算奖励、权益，并根据 `_isUnbound` 参数决定是否添加待解除绑定记录或从 `totalUnWithdrawn` 中扣除权益。
   - 调用 `subTotalStake` 减少总权益及触发排名操作。
   - 返回排名操作结果。
10. **委托人领取奖励 (`delegatorClaimAny`)**
    - 调用 `handleDelegatorPunishment` 处理惩罚。
    - 计算委托人的staking奖励，并重置委托人的债务和已结算奖励。
    - 计算可提取的解除绑定权益（`_unboundAmount`）。
    - 若验证者处于退出状态且解锁期已过，将剩余权益强制解除绑定（`_forceUnbound`），并从总权益中移除。
    - 减少 `totalUnWithdrawn` 并根据实际奖励金额向委托人发送以太币，触发 `RewardsWithdrawn` 事件。
    - 返回可提取的解除绑定权益和强制解除绑定的权益。
11. **处理委托人惩罚 (`handleDelegatorPunishment`)**
    - 计算委托人应受的惩罚（`calcDelegatorPunishment`）。
    - 更新委托人的 `punishFree` 值。
    - 若惩罚金额大于0，先从委托人的权益中削减，若不足则从待解除绑定权益中削减。
12. **计算委托人惩罚 (`calcDelegatorPunishment`)**
    - 根据当前累计惩罚因子与委托人上次惩罚后的 `punishFree` 值计算应削减的权益比例。
    - 根据比例和委托人的总权益（包括待解除绑定部分）计算实际削减金额。
13. **判断是否允许staking操作 (`canDoStaking`)**
    - 返回验证者当前状态是否为允许staking的状态（如空闲、就绪或处于监禁状态但超过监禁期）。
14. **添加待解除绑定记录 (`addUnboundRecord`)**
    - 将指定委托人和解除绑定权益金额添加至 `UnboundRecord` 结构体中，设置解除锁定结束时间。
15. **处理可提取的解除绑定权益 (`processClaimableUnbound`)**
    - 遍历委托人的待解除绑定记录，释放已过解锁期的权益，返回可提取的解除绑定权益总额。
16. **从待解除绑定权益中削减 (`slashFromUnbound`)**
    - 接收一个地址 `_owner` 和一个金额 `_amount`，从该地址的待解除绑定记录中削减指定金额的权益。
    - 验证待解除绑定总额是否足够，然后遍历待解除绑定记录，依次扣减金额直至达到 `_amount` 或所有记录被处理。
    - 清理已完成的解除绑定记录，并更新待解除绑定总额。
17. **增加总权益并触发排名操作 (`addTotalStake`)**
    - 增加总权益（`totalStake`）和未提取权益总额（`totalUnWithdrawn`）。
    - 根据增加后总权益值调整验证者状态：
      - 如果达到阈值且自质押满足最小要求，则从非准备状态（Idle或Jail）变为准备状态（Ready），返回排名操作类型 `Up`。
      - 如果原为监禁状态（Jail），但总权益仍低于阈值或自质押不足最小要求，则变为闲置状态（Idle），返回 `Noop`。
18. **减少总权益并触发排名操作 (`subTotalStake`)**
    - 减少总权益（`totalStake`）。
    - 根据减少后总权益值调整验证者状态：
      - 如果原为准备状态（Ready），且总权益低于阈值，则变为闲置状态（Idle），返回排名操作类型 `Remove`；否则返回 `Down`。
      - 如果原为监禁状态（Jail），则检查总权益和自质押是否满足恢复到准备状态的条件，否则保持在闲置状态。返回相应排名操作类型。
19. **惩罚 (`punish`)**
    - 接收一个惩罚因子 `_factor`，根据此因子削减验证者和管理员的权益。
    - 更新累积惩罚因子（`accPunishFactor`），记录惩罚发生时的区块号（`punishBlk`），并将验证者状态设为监禁（`State.Jail`），触发 `StateChanged` 事件。
20. **查询可提取权益 (`anyClaimable`)**
    - 根据调用者身份（管理员或普通委托人）分别调用 `validatorClaimable` 或 `delegatorClaimable`，返回可提取的解除绑定权益和staking奖励。
21. **查询验证者可提取权益 (`validatorClaimable`)**
    - 计算验证者可提取的解除绑定权益和staking奖励（包括佣金），并按系数调整为实际值。
22. **查询委托人可提取权益 (`delegatorClaimable`)**
    - 计算委托人可提取的解除绑定权益和staking奖励，考虑惩罚因素（削减权益）、验证者退出状态及锁定期限。
23. **获取可提取解除绑定权益 (`getClaimableUnbound`)**
    - 用于计算指定地址 `_owner` 当前可提取的解除绑定权益。
24. **获取待解除绑定记录详情 (`getPendingUnboundRecord`)**
    - 返回指定地址 `_owner` 的待解除绑定记录列表中指定索引 `_index` 的记录所包含的权益金额和解锁结束时间。
25. **获取所有委托人数量 (`getAllDelegatorsLength`)**
    - 返回 `allDelegatorAddrs` 列表中委托人的数量。