## 概述

跨链通信协议（IBC）是一种互操作性协议，采用两层式设计：一层是传输层，负责将数据组装成通用数据包，以及跨链验证该数据；另一层是应用层，负责解读数据包内容，并决定对所传输数据应执行的下一步操作。IBC 协议在事务验证及数据包正确性方面的安全性是通过传输层的轻客户端来保证的，而不是通过负责在不同链上的 IBC 客户端之间传递数据包的中继器。

## IBC 架构

IBC 是个分层协议，主要由以下组成

- IBC/TAO
- IBC/APP

![IBC 架构](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*qloucLA9bXYnTGk8M3Jbrg.png)

### IBC/TAO

IBC/TAO 是一个以可靠和经过身份验证的方式在两个区块链之间中继数据包的层。

可靠指源连的数据包发送只会被目标链接受一次且无需信任，数据包是由Relayer在区块链之间进行传递，任何人都可以成为Relayer。

身份认证可以这么理解，ibc 的channel+port 定位了数据包要发送到哪个具体模块（合约）进行处理，任何其他的模块或者合约都无法通过通道发送。所以ibc的核心主要有以下部分：

- ligtht client 
- connection 
- channel
- packet

#### light client 

每当一个新的块头被附加到dst区块链上时，relayer就会查询该头并将其提交给src链上的IBC/TAO模块。然后IBC/TAO模块根据对方区块链执行轻客户端协议以检查块头是否有效，并且只有在块头有效的情况下才更新dst区块链的*ClientState*。

一旦*ClientState*被更新为具有dst链的最新块头，IBC/TAO模块就可以检查所呈现的状态是否存储在对方区块链中。例如，如果dst链使用merkle树来存储其世界状态，并在每个区块头中包括merkle树根哈希（如以太坊），则可以使用merkle-证明来证明和验证树中的智能合约状态存在于区块链上. 示意图如下：

![彼此的轻客户端](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*ddIpls7yqJYqqdoAiimqhQ.png)


#### connection
两条链之间的连接状态转换都是由Relayer完成的.

阶段1：connOpenInit

INIT状态的新连接被创建并存储在启动区块链上

阶段2：connOpenTry

如果已验证INIT状态的连接是在发起链上创建的，relyer则会创建TRYOPEN状态的新连接并将其存储在相反的区块链上。

阶段3：connOpenAck

如果src链已验证TRYOPEN状态的连接是在dst链上创建的，则src链上的连接状态将从INIT更新为OPEN	

阶段4：connOpenConfirm

如果验证了发起链上的连接状态从INIT转换为OPEN，则在相反链上，连接状态将从TRYOPEN更新为OPEN

![connection process](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*uR79CCsMGfsk7h-CkIkr2g.png)

#### channel
通道和连接绑定，用来传输跨链数据包，通道的状态转换也是跟connection一样的，IBC会根据channel+port声明其跨链能力，比如某个APP模块具备跨链能力。和不同的链进行跨链则会创建不同的channel，不同的跨链APP则会声明不同的能力，这种设计保证了跨链的安全和稳定。

#### packet
packet 代表着用户从应用层发出跨链交易，通过IBC模块转换成packet，Relayer订阅事件扫到相关sendpacket事件后，会构建msgRecvPacket在dst链进行处理。其生命周期如下：

![pack flow](https://ibcprotocol.dev/_nuxt/img/packet_flow.1d89ee0.png)

正常流程：

用户通过APP A 发送跨链包给ibc模块，relayer 查询跨链包并生成msgRecvPacket,在链B调用，通过IBC B的OnRecvPacket回调App B，APP B 返回ACK 给Relayer,Relayer生成AckPacket在链A调用，IBC A通过 onAcknowLedgePacket 回调APP A处理ACK，比如转账，失败的ACK会从取消托管账户的资金。

超时流程：

用户通过APP A 发送跨链包给ibc模块，relayer 查询跨链包并生成msgRecvPacket,Relayer查询相关包receipt，如果在规定的timeout时间内没有，则构造timeoutPacket发往src链，src会根据此消息做出超时处理。


## 整体过程

user 基于IBC/APP 发送跨链交易，IBC/TAO将其转换成packet，并记录在相关存储中。

relayer 通过事件监听，将IbcmessageEvent读取到,这里是*EventTypeSendPacket*， 他都会转换成msg,扔到缓存里，然后根据配置的路径合并消息并进行相应的处理

接着realyer准备中继packets

- relayer对 packet 进行基础校验	

- Relayer向src链查询packect commitment(packet proof，proof包括root hash, proof path,以及要验证的key值) 以及对应高度

- 然后准备一个msgrecvPackect 在 dst 上带调用 (附带了src packet 的包的存在性证明)

  ```go
  msg := &chantypes.MsgRecvPacket{
  		Packet:          msgTransfer.Packet(),
  		ProofCommitment: proof.Proof,
  		ProofHeight:     proof.ProofHeight,
  		Signer:          signer,
  	}
  ```

- 准备升级dst链上src的客户端的消息到最新（也是为了轻客户端验证）（实际是从src链查询的，有可能作恶，但是会被dst上面的更新客户端逻辑验证），relayer是获取了src最新的header，以及保存在dst上最新的header，分别进行验证，保存的验证直接通过store,src最新的header验证。

- 将消息发到对应链的池子中 

接着dst上的链该要执行msgrecvPacket了

- 第一步是IBC/TAO层的验证，dst链必须有能力处理这个包，同时确保包不是乱发的，且和对手链之间已经建立了连接和通道，并且从未接受过这个packet，最重要的一点是要证明src的确有这个packet信息，最后需要设置已经接收了这个packet的receipt
- ibc 执行回调函数OnRecvPacket（app注册的实际接收packet的处理），执行成功与否会返回ack,代表了相关信息
- dst链设置ack ,为了在src链上的dst客户端可以验证

接着又是relyer登场，他会监听write_acknowledgement 事件进行处理

- 查询dst链上ack的proof，确认在dst链上发生
- 构造MsgAcknowledgement消息
- relayer把dst最新的客户端状态更新到src上
- src链对ack包进行验证(是否是dst链发过来的)， 删除数据包承诺，由于数据包已被确认，因此不再需要该承诺

到此整个跨链过程结束。

## 存在性证明

在跨链交互中，涉及到了多个类型的消息， 比如channel，connection，packet等消息，Relayer在传递消息的过程中，也要将消息对应的存在性证明带给对手链，这样对手链可以根据源连的轻客户端以及证明来验证消息的正确性。其主要原理是基于IVAL+ tree的proof验证以及proof生成。

IAVL+树是一种有序键的树结构，其中每个节点都代表一个键值对，节点内部存储键的值以及子树的元数据。IAVL+树的每个节点都有一个哈希值，这个哈希值是通过节点的键值、子树哈希值和节点元数据计算得出的。

现在假设我们要验证某个键K是否存在于IAVL+树中。我们可以通过树的根哈希值（roothash）和键K来计算出一个证明路径，这个证明路径包含了树中所有涉及到K的节点，包括K所在的叶子节点和所有的祖先节点。具体步骤如下：

1. 从根节点开始，将K与当前节点存储的键进行比较。
2. 如果K等于当前节点存储的键，则停止搜索，返回证明路径。如果K小于当前节点存储的键，则进入当前节点的左子树继续搜索；如果K大于当前节点存储的键，则进入当前节点的右子树继续搜索。
3. 在搜索过程中，每个节点都将其子树哈希值加入到证明路径中。
4. 当搜索到K所在的叶子节点时，将叶子节点的哈希值也加入到证明路径中。
5. 最终，证明路径中包含了树中所有涉及到K的节点的哈希值。

通过证明路径，我们可以验证K是否存在于IAVL+树中。具体的验证步骤如下：

1. 从根哈希值开始，计算每个涉及到K的节点的哈希值，直到计算出K所在的叶子节点的哈希值。
2. 将证明路径中所有节点的哈希值按照它们在树中的位置排序，并将它们合并成一个单一的哈希值。
3. 将这个合并后的哈希值与根哈希值进行比较。如果两个哈希值相等，则K存在于树中；否则，K不存在于树中。

## 跨链转账举例

参考自ICS-20 ，支持跨链传输token，以从A链到B链为例：

transfer from A to B ：

1. ChainA以原子方式执行以下操作：

- 锁定ICS-20模块中的令牌

- 然后对指定锁定代币数量和面额的数据包执行*sendPacket*操作

2. ChainB以原子方式执行以下操作：

- 对数据包执行*recvPacket*操作

- 然后铸造等同于ICS-20模块中锁定代币的代金券代币

3. ChainA正常执行*acknowledgePacket*。

![ics-20](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*lMD8xBxM26Cs6XTVNLCEUw.png)

-------

back from B to A:

1. ChainB以原子方式执行以下操作：

- 在ICS-20模块中焚烧代金券代币
- 然后对指定烧毁凭证金额和面额的数据包执行*sendPacket*操作

2. ChainA以原子方式执行以下操作：

- 对数据包执行*recvPacket*操作

- 然后解锁相当于ICS-20模块中烧毁的代金券的代币

3. ChainB正常执行*acknowledgePacket*。

![back](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*zWK3lSRBjyonxO-KAaJoVw.png)

​	整体还是基于 lock+mint+ 托管+轻客户端验证模型，算是比较稳妥的技术方案。

--------

