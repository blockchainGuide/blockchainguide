> 翻译自：https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter

# 1. 介绍

Filecoin是一个基于区块链机制的分布式存储网络。Filecoin矿工可以选择为网络提供存储容量，从而通过定期生成加密证明来获得Filecoin加密货币(FIL)的单位，证明他们正在提供指定的容量。此外，Filecoin允许各方通过Filecoin区块链上的共享分类账记录的交易来交换Filecoin货币。然而，Filecoin并没有使用工作量证明来维护链上的共识，而是使用了存储证明本身:一个矿工在共识协议中的能力与它提供的存储量成比例。

Filecoin区块链不仅维护Filecoin交易和账户的分类账本，还实现了Filecoin VM，这是一个可复制的状态机，在网络上的参与者之间执行各种加密合约和市场机制。这些合约包括存储交易，客户向矿工支付FIL货币，以换取存储客户请求的特定文件数据。通过Filecoin VM的分布式实现，存储交易和其他记录在链上的合约机制将随着时间的推移继续处理，而不需要原始各方(如请求数据存储的客户端)的进一步交互。



## 1.2 架构图

Actor状态图：

![image-20210201094233728](https://tva1.sinaimg.cn/large/008eGmZEgy1gn7sf9164mj31r10u0jza.jpg)

## 1.3 关键内容

- **Data structures**是带有语义标记的数据成员(例如，结构、接口或枚举)的集合。
- **Function**是不依赖于外部状态的计算过程(例如，数学函数或不引用全局变量的编程语言函数)
- **Components**是一组功能，在实现结构中被表示为单个软件单元。根据语言和特定组件的选择，这可能对应于单个软件模块、运行主循环的线程或进程、磁盘支持的数据库或各种其他设计选择。例如，ChainSync是一个组件:它可以被实现为一个进程或线程运行单个指定的主循环，它等待网络消息，并通过记录和/或转发块数据来响应。
-  **api**是将消息传递给组件的接口。一个给定子协议的客户端视图，例如向一个miner节点的Storage Provider组件请求在存储市场中存储文件，可能需要执行一系列的API请求。
- **Nodes**是与协议交互的完整的软件和硬件系统。一个节点可能会不断地运行上述几个组件，参与多个子系统，并在本地和/或通过网络公开api，这取决于节点配置。术语“完整节点”是指运行上述所有组件并支持规范中详细描述的所有api的系统。
- **Subsystems**是整个Filecoin协议的概念划分，要么是按照完整的协议(如存储市场或检索市场)，要么是按照功能(如VM - Virtual Machine)。它们不一定对应于任何特定的节点或软件组件。
- **Actor**是体现在Filecoin VM状态中的虚拟实体。协议参与者类似于智能合约的参与者;参与者携带FIL货币余额，并可以通过VM的操作与其他参与者交互，但并不一定对应于任何特定的节点或软件组件。

## 1.4 FileCoin VM 

 Filecoin的大多数用户所面对的功能(支付、存储市场、能量表等)是通过Filecoin虚拟机(Filecoin VM)管理的。该网络生成一系列区块，并同意哪个区块链是正确的。每个块包含一系列称为消息的状态转换，以及应用这些消息后的当前全局状态检查点。

这里的全局状态由一组参与者（**Actor**）组成，每个参与者都有自己的私有状态。

 一个参与者是Filecoin等价于以太坊的智能合约，它本质上是Filecoin网络中的一个“对象”，具有状态和一组可以用来与之交互的方法。每个参与者都有一个Filecoin balance，一个状态指针，一个代码CID(告诉系统参与者是什么类型的参与者)，以及一个nonce(跟踪参与者发送的消息数量)。

在参与者上调用方法有两种途径。首先，要作为系统的外部参与者(也就是使用Filecoin的普通用户)调用方法，您必须向网络发送一个签名消息，并向包含您的消息的矿工支付费用。消息上的签名必须与一个帐户相匹配，该帐户有足够的Filecoin来支付消息的执行费用。这里的费用相当于比特币和以太坊的交易费用，它与处理消息所做的工作成比例(比特币按字节为消息定价，以太坊使用“gas”的概念。我们也用gas)。

其次，参与者可以在调用其方法之一的过程中调用另一个参与者的方法。然而，唯一可能发生这种情况的时候是某个参与者被外部用户消息调用的结果(注意:用户调用的参与者可能调用另一个参与者，然后调用另一个参与者，在执行能够承受的层次上尽可能多)。

具体实现请参见虚拟机子系统。

## 1.5 系统分解

### 1.5.1 系统是什么，如何工作

Filecoin将功能解耦并模块化到松散连接的系统中。每个系统都增加了重要的功能，通常是为了实现一组重要且紧密相关的目标。

例如，区块链系统提供了块、Tipset和链等结构，并提供了块同步、块传播、块验证、链选择和链访问等功能。这与文件、片段、片段准备和数据传输是分开的。这两个系统都与市场分离，市场提供订单、交易、市场可见性和交易结算。

#### 1.5.1.1 为什么系统解耦有用?

理由如下：

- **实施边界：**可以构建仅实施一部分系统的Filecoin实施。这对于*实现多样性*特别有用：我们需要许多安全关键系统（例如，区块链）的实现，但不需要许多可以分离的系统实现。
- **运行时解耦：**系统解耦使构建和运行将系统隔离到单独程序甚至单独物理计算机中的Filecoin节点变得更加容易。
- **安全隔离：**某些系统比其他系统需要更高的操作安全性。系统解耦使实现能够满足其安全性和功能性需求。一个很好的例子是将区块链处理与数据传输分开。
- **可伸缩性：**系统和各种用例可能会为不同的操作员带来不同的性能要求。系统解耦使运营商更容易沿着系统边界扩展其部署。

#### 1.5.1.2 FileCoin 节点不需要所有系统

- **实施边界：**可以构建仅实施一部分系统的Filecoin实施。这对于*实现多样性*特别有用：我们需要许多安全关键系统（例如，区块链）的实现，但不需要许多可以分离的系统实现。
- **运行时解耦：**系统解耦使构建和运行将系统隔离到单独程序甚至单独物理计算机中的Filecoin节点变得更加容易。
- **安全隔离：**某些系统比其他系统需要更高的操作安全性。系统解耦使实现能够满足其安全性和功能性需求。一个很好的例子是将区块链处理与数据传输分开。
- **可伸缩性：**系统和各种用例可能会为不同的操作员带来不同的性能要求。系统解耦使运营商更容易沿着系统边界扩展其部署。

#### 1.5.1.3 分离系统

> 我们如何确定一个系统与另一个系统属于什么功能？

在系统之间划定界限是将紧密相关的功能与无关部分分开的技术。从某种意义上说，我们寻求将紧密集成的组件保留在同一系统中，并远离其他无关的组件。有时这很简单，边界自然是来自数据结构或功能。例如，很容易观察到，客户和矿工彼此协商交易与VM执行非常无关。

有时这比较困难，并且需要整理，添加或删除抽象。例如，`StoragePowerActor`和和以前`StorageMarketActor`是单个`Actor`。这导致了整个`StorageDeal`生产`StorageMarket`市场，存储市场，部门密封，PoSt生成等功能之间的巨大耦合。纠结这两组相关功能需要将一个参与者分成两个。

#### 1.5.1.4 在系统内部分解

系统本身分解为较小的子单元。这些有时称为“子系统”，以避免与更大的一流系统混淆。子系统本身可能会进一步崩溃。此处的命名未严格执行，因为这些细分与协议和实现工程方面的问题相比，与用户功能更相关。

### 1.5.2 实现系统

#### 1.5.2.1 系统要求

为了更轻松地将功能分离到系统中，Filecoin协议假定了一组可用于所有系统的功能。此功能可以通过各种方式的实现来实现，并且应将此处的指南作为建议（应该）。

本文档中定义的所有系统都要求具备以下条件：

- 仓库：
  - **本地的`IpldStore`。**用于数据结构（小型结构化对象）的一定数量的持久本地存储。系统期望使用IpldStore进行初始化，在该系统中存储它们希望在崩溃中持续存在的数据结构。
  - **用户配置值。**少量用户可编辑的配置值。这些应该使最终用户易于访问，查看和编辑。
  - **本地，安全`KeyStore`。**用于生成和使用加密密钥的工具，必须对Filecoin节点保密。系统不应直接访问密钥，而应通过`KeyStore`提供加密，解密，签名，SigVerify等功能的抽象（即）进行访问。
- **本地的`FileStore`。**某些文件的持久本地存储（大字节数组）。系统期望使用存储大文件的FileStore进行初始化。某些系统（例如Markets）可能需要存储和删除大量较小的文件（1MB-10GB）。其他系统（例如存储挖掘）可能需要存储和删除大量大文件（1GB-1TB）。
- **网络。**大多数系统需要访问网络，才能连接到其他Filecoin节点中的对应系统。系统期望使用可`libp2p.Node`在其上安装自己的协议的进行初始化。
- **时钟。**有些系统需要访问当前的网络时间，而有些系统的漂移容差较低。系统期望使用一个时钟来初始化，以从中得知网络时间。一些系统（如区块链）需要很少的时钟漂移，并且需要*安全的*时间。

为此，我们使用`FilecoinNode`数据结构，该数据结构在初始化时传递给所有系统。

#### 1.5.2.2系统限制

此外，系统必须遵守以下限制：

- **随机崩溃。**Filecoin节点可能随时崩溃。通过崩溃，系统必须是安全且一致的。这主要是通过限制持久状态的使用，通过Ipld数据结构持久化这种状态，以及通过使用检查状态的初始化例程以及可能纠正错误来实现的。
- **隔离。**系统必须通过定义良好的隔离接口进行通信。他们不得在共享内存空间上构建关键功能。（注意：为了提高性能，共享内存抽象可用于为IpldStore，FileStore和libp2p供电，但是系统本身不应该需要它。）它还显着简化了协议，并使其更易于理解，分析，调试和更改。
- **无法直接访问主机操作系统文件系统或磁盘。**系统无法直接访问磁盘，而是通过FileStore和IpldStore抽象来进行。这将为最终用户（尤其是存储矿工和大量数据的客户端）提供高度的可移植性和灵活性，这需要能够轻松替换其Filecoin节点访问本地存储的方式。
- **不能直接访问主机OS网络堆栈或TCP / IP。**系统无法直接访问网络-它们通过libp2p库进行访问。不得有任何其他类型的网络访问。这提供了跨平台和网络协议的高度可移植性，使Filecoin节点（及其所有关键系统）可以使用各种协议（例如蓝牙，LAN等）在多种设置下运行。

# 2. 系统

在本节中，我们将逐一详细说明所有系统组件，以提高其复杂性和/或与其他系统组件的相互依赖性。组件之间的交互仅在适当的地方简要讨论，但总体工作流程在“简介”部分给出。特别是，在本节中，我们讨论：

- Filecoin节点：参与Filecoin网络的不同类型的节点，以及这些节点运行的重要部分和过程，例如密钥库和IPLD存储，以及libp2p的网络接口。
- 文件和数据：Filecoin的数据单位，例如“部门”和“碎片”。
- 虚拟机：Filecoin VM的子组件，例如参与者，即在Filecoin区块链上运行的智能合约和状态树。
- 区块链：Filecoin区块链的主要构建块，例如消息和块的结构，消息池，以及节点首次加入网络时如何同步区块链。
- 令牌：钱包所需的组件。
- 存储挖掘：存储挖掘的详细信息，存储能力共识以及存储矿工如何证明存储（不涉及证明的细节，这将在后面讨论）。
- 市场：存储和检索市场，主要是脱链的过程，但对于分散存储市场的平稳运行非常重要。

## 2.1 [Filecoin节点](https://spec.filecoin.io/#section-systems.filecoin_nodes)

本节首先讨论Filecoin节点的概念。尽管在Filecoin的Lotus实现中不像在其他区块链网络中那样严格定义不同的节点类型，但是不同类型的节点应该实现不同的属性和功能。简而言之，基于节点提供的*服务*集对其进行定义。

在本节中，我们还将讨论与在Filecoin节点中存储系统文件有关的问题。请注意，在本节中，通过存储，我们不是指节点为在网络中进行挖掘而提交的存储，而是指它需要可用于密钥和IPLD数据的本地存储库。

在本节中，我们还将讨论网络接口以及节点之间如何查找和连接，如何使用libp2p交互和传播消息以及如何设置节点的时钟。

### 2.1.1[节点类型](https://spec.filecoin.io/#section-systems.filecoin_nodes.node_types)

Filecoin网络中的节点主要根据其提供的服务进行标识。因此，节点的类型取决于节点提供的服务。Filecoin网络中的一组基本服务包括：

- 链验证
- 仓储市场客户
- 存储市场提供商
- 检索市场客户
- 检索市场提供者
- 仓储采矿

参与Filecoin网络的任何节点都应至少提供*链验证*服务。根据节点在链验证之上提供的额外服务，它会获得相应的功能和节点类型“标签”。

可以使用主机中的存储库（目录）以一对一关系实现节点-即，一个存储库属于单个节点。也就是说，一台主机可以通过具有相应的存储库来实现多个Filecoin节点。

Filecoin实现可以支持以下子系统或节点类型：

- **链验证器节点：**这是节点加入Filecoin网络所需的最低功能。除非实现以下所述的**客户端节点**功能，否则这种类型的节点无法在网络中发挥积极作用。链验证器节点首次加入网络时必须同步链（ChainSync），以达成当前共识。从那时起，该节点必须不断获取链中的任何附加内容（即，接收最新的块）并验证它们是否达到共识状态。
- **客户端节点：**这种类型的节点建立在**Chain Verifier节点**之上，并且必须由Filecoin网络上构建的任何应用程序来实现。可以将其视为基于Filecoin的应用程序（例如交易所或分散存储应用程序）的主要基础结构节点（至少就与区块链的交互而言）。该节点应实现*存储市场和检索市场客户*服务。客户端节点应与存储和检索市场进行交互，并能够通过数据传输模块进行数据传输。
- **Retrieval Miner Node（检索矿工节点）：**此节点类型扩展了**Chain Verifier节点**以添加*检索矿工*功能，即参与了检索市场。这样，此节点类型需要实现*检索市场提供者*服务，并能够通过数据传输模块进行数据传输。
- **Storage Miner Node：**这种类型的节点必须实现验证，创建和添加块以扩展区块链所需的所有必需功能。它应实施链验证，存储挖掘和存储市场提供商服务，并能够通过数据传输模块进行数据传输。

#### 2.1.1.1[ 节点接口](https://spec.filecoin.io/#section-systems.filecoin_nodes.node_types.node-interface)

可以在[此处](https://github.com/filecoin-project/lotus/blob/master/node/repo/interface.go)找到Node接口的Lotus实现 。

#### 2.1.1.2[链验证器节点](https://spec.filecoin.io/#section-systems.filecoin_nodes.node_types.chain-verifier-node)

```go
type ChainVerifierNode interface {
  FilecoinNode

  systems.Blockchain
}
```

可以在[这里](https://github.com/filecoin-project/lotus/blob/master/node/impl/full.go)找到Chain Verifier Node的Lotus实现 。

#### 2.1.1.3[客户端节点](https://spec.filecoin.io/#section-systems.filecoin_nodes.node_types.client-node)

```go
type ClientNode struct {
  FilecoinNode

  systems.Blockchain
  markets.StorageMarketClient
  markets.RetrievalMarketClient
  markets.DataTransfers
}
```

客户端节点的Lotus实现可以在[这里](https://github.com/filecoin-project/lotus/blob/master/node/impl/client/client.go)找到 。

#### 2.1.1.4[存储矿工节点](https://spec.filecoin.io/#section-systems.filecoin_nodes.node_types.storage-miner-node)

```go
type StorageMinerNode interface {
  FilecoinNode

  systems.Blockchain
  systems.Mining
  markets.StorageMarketProvider
  markets.DataTransfers
}
```

可以在[此处](https://github.com/filecoin-project/lotus/blob/master/node/impl/storminer.go)找到Storage Miner Node的Lotus实现 。

#### 2.1.1.5[检索矿工节点](https://spec.filecoin.io/#section-systems.filecoin_nodes.node_types.retrieval-miner-node)

```go
type RetrievalMinerNode interface {
  FilecoinNode

  blockchain.Blockchain
  markets.RetrievalMarketProvider
  markets.DataTransfers
}
```

#### 2.1.1.6[中继节点](https://spec.filecoin.io/#section-systems.filecoin_nodes.node_types.relayer-node)

```go
type RelayerNode interface {
  FilecoinNode

  blockchain.MessagePool
}
```

#### 2.1.1.7[节点配置](https://spec.filecoin.io/#section-systems.filecoin_nodes.node_types.node-configuration)

可以在[此处](https://github.com/filecoin-project/lotus/blob/master/node/config/def.go)找到Filecoin Node配置值的Lotus实现 。

### 2.1.2[节点存储库](https://spec.filecoin.io/#section-systems.filecoin_nodes.repository)

Filecoin节点存储库只是系统和链数据的本地存储。它是任何功能性Filecoin节点需要在本地存储以便正确运行的数据的抽象。

该存储库可供节点的系统和子系统访问，并且可以从节点的存储区分开`FileStore`。

该存储库存储节点的密钥，有状态对象的IPLD数据结构以及节点配置设置。

FileStore存储库的Lotus实现可以在[这里](https://github.com/filecoin-project/lotus/blob/master/node/repo/fsrepo.go)找到 。

#### 2.1.2.1[密钥库](https://spec.filecoin.io/#section-systems.filecoin_nodes.repository.key_store)

这`Key Store`是任何完整Filecoin节点中的基本抽象，用于存储与给定矿工的地址（请参阅下面的实际定义）和不同的工作程序（矿工应该选择运行多个工作程序）关联的密钥对。

节点安全性在很大程度上取决于保持这些密钥的安全性。为此，我们强烈建议：1）将密钥与所有子系统分开，2）根据其他子系统的要求使用单独的密钥存储来签署请求，以及3）保留未用作冷库中挖掘的那些密钥。

Filecoin存储矿工依赖三个主要组成部分：

- 在调用`registerMiner()`Storage Power Consensus子系统后，**将存储矿工*****参与者\*****地址**唯一地分配给给定的存储矿工**参与者****地址**。实际上，存储矿工本身没有地址，而是由与其绑定的参与者的地址来标识的。这是给定存储矿工的唯一标识符，其电源和其他密钥将与之关联。该`actor value`指定一个已经创建的矿工演员的地址。
- **所有者密钥对**由矿工在注册之前提供，并且其公钥与矿工地址相关。所有者密钥对可用于管理矿工和提取资金。
- **工人密钥对**是与存储矿工参与者地址关联的公共密钥。可由矿工选择和更改。辅助密钥对用于签名块，也可以用于签名其他消息。鉴于它是[可验证随机函数的](https://spec.filecoin.io/#section-algorithms.crypto.vrf)一部分，它必须是BLS密钥对 。

多个存储矿工参与者可以共享一个所有者公共密钥，也可以共享一个工人公共密钥。

在[Storage Miner Actor中](https://spec.filecoin.io/#section-systems.filecoin_mining.storage_mining.storage_miner_actor)指定了更改链上工作程序密钥对（即与存储矿工actor关联的工作人员Key）的过程 。请注意，这是一个两步过程。首先，矿工通过向链发送消息来进行更改。然后，矿工在随机回溯时间之后确认密钥更改。最后，在额外的随机回溯时间之后，矿工将开始使用新密钥对块进行签名。存在此延迟是为了防止自适应密钥选择攻击。

密钥安全在Filecoin中至关重要，每个区块链中的密钥也是如此。**无法安全地存储和使用密钥或将私钥暴露给对手可能会导致对手有权使用矿工的资金。**

#### 2.1.2.2[IPLD商店](https://spec.filecoin.io/#section-systems.filecoin_nodes.repository.ipldstore)

星际链接数据（IPLD）是一组库，这些库允许跨不同分布式系统和协议的内容寻址数据结构互操作。它为原始密码哈希提供了一种基本的“通用语言”，使数据结构可以在两个独立的协议之间被可验证地引用和检索。例如，用户可以在以太坊交易或智能合约中引用IPFS目录。

Filecoin节点的IPLD存储是用于散列链接数据的本地存储。

IPLD基本上由三层组成：

- 块层，着重于块格式和寻址，块如何广告或自描述其编解码器
- 数据模型层，它定义了一组必须包含在任何实现中的必需类型-下文将详细讨论。
- 模式层，它允许扩展数据模型以与更复杂的结构进行交互，而无需自定义转换抽象。

有关IPLD的更多详细信息，请参见其 [规范](https://github.com/ipld/specs)。

##### 2.1.2.2.1[数据模型](https://spec.filecoin.io/#section-systems.filecoin_nodes.repository.ipldstore.the-data-model)

IPLD的核心是定义用于表示数据的 [数据模型](https://github.com/ipld/specs/blob/master/data-model-layer/data-model.md)。数据模型旨在通过各种编程语言进行实际实现，同时保持对内容寻址数据的可用性以及与该数据交互的各种通用工具。

数据模型包括一系列标准基本类型（或“种类”），例如布尔，整数，字符串，空值和字节数组，以及两种递归类型：列表和映射。由于IPLD是为内容寻址数据而设计的，因此IPLD在其数据模型中还包含“链接”原语。实际上，链接使用 [CID](https://github.com/multiformats/cid)规范。IPLD数据被组织为“块”，其中一个块由原始编码数据及其内容地址或CID表示。每个内容可寻址的数据块都可以表示为一个块，并且块可以一起形成一个相干图或 [Merkle DAG](https://docs.ipfs.io/guides/concepts/merkle-dag/)。

应用程序通过数据模型与IPLD交互，而IPLD通过一组编解码器处理编组和解组。IPLD编解码器可能支持完整的数据模型或部分数据模型。支持完整数据模型的两个编解码器是 [DAG-CBOR](https://github.com/ipld/specs/blob/master/block-layer/codecs/dag-cbor.md)和 [DAG-JSON](https://github.com/ipld/specs/blob/master/block-layer/codecs/dag-json.md)。这些编解码器分别基于CBOR和JSON序列化格式，但包括允许它们封装IPLD数据模型（包括其链接类型）的规范化以及可在任何数据集及其各自的内容地址（或哈希摘要）。这些规则包括在对地图进行编码时对键的特定顺序进行规定，或者在存储时对整数类型进行大小调整。

##### 2.1.2.2.2 [Filecoin中的IPLD](https://spec.filecoin.io/#section-systems.filecoin_nodes.repository.ipldstore.ipld-in-filecoin)

Filecoin网络中有两种方式使用IPLD：

- 所有系统数据结构都使用DAG-CBOR（IPLD编解码器）存储。DAG-CBOR是CBOR的更严格子集，具有预定义的标记方案，旨在存储，检索和遍历散列链接的数据DAG。与CBOR相比，DAG-CBOR可以保证确定性。
- Filecoin网络上存储的文件和数据也使用各种IPLD编解码器（不一定是DAG-CBOR）进行存储。

IPLD在数据上方提供了一致且一致的抽象，从而使Filecoin可以构建复杂的多块数据结构（例如HAMT和AMT）并与之交互。Filecoin使用DAG-CBOR编解码器对其数据结构进行序列化和反序列化，并使用IPLD数据模型与该数据进行交互，并在此模型上构建了各种工具。IPLD选择器还可用于寻址链接数据结构中的特定节点。

###### 2.1.2.2.2.1[IpldStores](https://spec.filecoin.io/#section-systems.filecoin_nodes.repository.ipldstore.ipldstores)

Filecoin网络主要依赖于两个不同的IPLD GraphStore：

- 一种`ChainStore`存储区块链的数据，包括块头，相关消息等。
- 一种`StateStore`存储来自给定`stateTree`区块链的有效负载状态，或者存储由[Filecoin VM](https://spec.filecoin.io/#section-systems.filecoin_vm)应用于给定状态的给定链中所有块消息的 [结果](https://spec.filecoin.io/#section-systems.filecoin_vm)。

的`ChainStore`是通过从他们的同辈节点中的引导阶段下载 [链同步](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync)，并且由节点此后被存储。每次接收到新的块时，或者节点同步到新的最佳链时，它都会更新。

的`StateStore`是通过在给定的所有块消息的执行计算`ChainStore`，并且由节点此后被存储。[VM解释器会](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter)使用每个新传入块的处理对其进行更新 ，并相应地由在[块标题](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block) `ParentState`字段中在其上方生成的新块进行引用 。

### 2.1.3 网络接口

Filecoin节点使用libp2p网络堆栈的几种协议来进行对等方发现，对等方路由以及块和消息传播。Libp2p是用于点对点网络的模块化网络堆栈。它包括多种协议和机制，可实现高效，安全和有弹性的对等通信。Libp2p节点彼此之间打开连接，并在同一连接上安装不同的协议或流。在最初的握手中，节点交换它们各自支持的协议，所有与Filecoin相关的协议都将安装在`/fil/...`协议标识符下。

libp2p的完整规范可以在[https://github.com/libp2p/specs中](https://github.com/libp2p/specs)找到 。这是Filecoin使用的libp2p协议的列表。

- **Graphsync：** Graphsync是一种用于在对等点之间同步图的协议。它用于在Filecoin节点之间引用，寻址，请求和传输区块链和用户数据。所述 [GraphSync的草案规范](https://github.com/ipld/specs/blob/master/block-layer/graphsync/graphsync.md)提供的概念，接口和由GraphSync使用的网络消息的更多细节。协议ID没有Filecoin特定的修改。
- **Gossipsub：**使用基于Gossip的pubsub协议（缩写为*GossipSub），*通过Filecoin网络传播块头和消息。与传统的pubsub协议一样，节点订阅主题并接收在这些主题上发布的消息。当节点从其订阅的主题接收消息时，它们将运行验证过程，并且i）将消息传递给应用程序； ii）将消息进一步转发给他们知道已订阅同一主题的节点。此外，FileCoin中使用的GossipSub v1.1版本通过安全机制进行了增强，该机制使协议可以抵御安全攻击。该 [GossipSub规格](https://github.com/libp2p/specs/tree/master/pubsub/gossipsub)提供有关其设计和实现的所有协议详细信息，以及协议参数的特定设置。尚未对协议ID进行filecoin的特定修改。但是话题标识必须是形式`fil/blocks/<network-name>`和`fil/msgs/<network-name>`
- **Kademlia DHT：** Kademlia DHT是一个分布式哈希表，在特定节点的最大查找数上具有对数范围。在Filecoin网络中，Kademlia DHT主要用于对等发现和对等路由。特别是，当一个节点想要在Filecoin网络中存储数据时，它们会获得一个矿工列表及其节点信息。该节点信息（除其他外）包括矿工的PeerID。为了连接到矿工并交换数据，想要在网络中存储数据的节点必须找到矿工的多地址，他们通过查询DHT来实现。所述 [libp2p喀DHT规范](https://github.com/libp2p/go-libp2p-kad-dht)提供了DHT结构的实现细节。对于Filecoin网络，协议ID的格式必须为`fil/<network-name>/kad/1.0.0`。
- **引导程序列表：**这是新节点在加入网络后尝试连接的节点列表。引导节点列表及其地址由用户（即应用程序）定义。
- **对等交换：**此协议是在Kademlia DHT上面讨论的对等发现过程的实现。通过与DHT进行接口，它使对等点可以找到网络中其他对等点的信息和地址，并为要连接的对等点创建和发出查询。

### 2.1.4[时钟](https://spec.filecoin.io/#section-systems.filecoin_nodes.clock)

Filecoin假定系统参与者之间的时钟同步较弱。也就是说，系统依赖于参与者可以访问全局同步时钟（允许某些有界偏移）。

Filecoin依靠此系统时钟来确保共识。具体来说，时钟是支持验证规则所必需的，验证规则可防止块生产者使用协议的时间戳来挖掘具有未来时间戳的块并更频繁地运行领导者选举。

#### 2.1.4.1[时钟用途](https://spec.filecoin.io/#section-systems.filecoin_nodes.clock.clock-uses)

使用Filecoin系统时钟：

- 通过同步节点来验证传入块是否在给定时间戳的适当纪元内被挖掘（请参见 [块验证](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block.block-syntax-validation)）。这是可能的，因为系统时钟始终将时间映射到唯一的纪元号，该纪元号完全由创世块中的开始时间确定。
- 通过同步节点以放置来自未来纪元的数据块
- 通过允许节点在下一轮尝试领导者选举（如果在当前轮中没有人产生阻碍的情况下）来挖掘节点，以保持协议的活跃性（请参阅 [存储电源共识](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus)）。

为了允许矿工执行上述操作，系统时钟必须：

1. 相对于其他节点具有足够低的偏移量，以使从其他节点的角度来看，不会在被认为是未来纪元的纪元中开采区块（这些区块直到根据[验证规则](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block.block-semantic-validation)的正确纪元/时间才被 [验证](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block.block-semantic-validation)）。
2. 设置节点初始化的时期数等于 `epoch = Floor[(current_time - genesis_time) / epoch_time]`

预计其他子系统将从`NewRound()`时钟子系统注册到事件。

#### 2.1.4.2[时钟要求](https://spec.filecoin.io/#section-systems.filecoin_nodes.clock.clock-requirements)

用作Filecoin协议一部分的时钟应保持同步，且偏移小于1秒，以便进行适当的验证。

预计计算机级晶体的偏差为 [1ppm](https://www.hindawi.com/journals/jcnc/2008/583162/)（即每秒1微秒，或每周0.6秒）。因此，为了遵守上述要求：

- 节点应运行NTP守护程序（例如timesyncd，ntpd，chronyd），以使其时钟与一个或多个可靠的外部引用保持同步。
  - 我们建议以下来源：
    - **`pool.ntp.org`**（ [详细](https://www.ntppool.org/en/use.html)）
    - `time.cloudflare.com:1234`（ [详细](https://www.cloudflare.com/time/)）
    - `time.google.com`（ [详细](https://developers.google.com/time)）
    - `time.nist.gov`（ [详细](https://tf.nist.gov/tf-cgi/servers.cgi)）
- 较大的采矿作业可能会考虑使用具有GPS参考和/或频率稳定的外部时钟的本地NTP / PTP服务器，以改善计时功能。

采矿业务有强烈的动机来防止时钟向前倾斜一个纪元以上，以防止块状提交被拒绝。同样，他们有动机防止时钟偏离一个以上的时间，以避免将自己与网络中的同步节点分开。

## 2.2[文件和数据](https://spec.filecoin.io/#section-systems.filecoin_files)

Filecoin的主要目的是存储客户的文件和数据。本节详细介绍与处理文件，分块，编码，图形表示`Pieces`，，存储抽象等相关的数据结构和工具。

### 2.2.1[文件](https://spec.filecoin.io/#section-systems.filecoin_files.file)

[例：](https://spec.filecoin.io/#example-)

```go
// Path is an opaque locator for a file (e.g. in a unix-style filesystem).
type Path string

// File is a variable length data container.
// The File interface is modeled after a unix-style file, but abstracts the
// underlying storage system.
type File interface {
    Path()   Path
    Size()   int
    Close()  error

    // Read reads from File into buf, starting at offset, and for size bytes.
    Read(offset int, size int, buf Bytes) struct {size int, e error}

    // Write writes from buf into File, starting at offset, and for size bytes.
    Write(offset int, size int, buf Bytes) struct {size int, e error}
}
```

#### 2.2.1.1[FileStore-文件的本地存储](https://spec.filecoin.io/#section-systems.filecoin_files.file.filestore)

的`FileStore`是用来指任何底层系统或设备，其将Filecoin其数据存储到一个抽象。它基于Unix文件系统语义，并包含的概念`Paths`。在这里使用这种抽象是为了确保Filecoin的实现使最终用户可以轻松地使用适合他们需求的基础替换底层存储系统。最简单的版本`FileStore`只是主机操作系统的文件系统。

[例：](https://spec.filecoin.io/#example-)

```go
// FileStore is an object that can store and retrieve files by path.
type FileStore struct {
    Open(p Path)           union {f File, e error}
    Create(p Path)         union {f File, e error}
    Store(p Path, f File)  error
    Delete(p Path)         error

    // maybe add:
    // Copy(SrcPath, DstPath)
}
```

##### 2.2.1.1.1[变化的用户需求](https://spec.filecoin.io/#section-systems.filecoin_files.file.filestore.varying-user-needs)

Filecoin用户的需求差异很大，许多用户（尤其是矿工）将在Filecoin的下方和周围实施复杂的存储架构。`FileStore`这里的抽象是为了使这些变化的需求易于满足。Filecoin协议中的所有文件和扇区本地数据存储都是通过此`FileStore`接口定义的，这使实现易于实现可交换，并且使最终用户可以轻松选择所选择的系统。

##### 2.2.1.1.2[实施实例](https://spec.filecoin.io/#section-systems.filecoin_files.file.filestore.implementation-examples)

该`FileStore`接口可以由多种后备数据存储系统来实现。例如：

- 主机操作系统文件系统
- 任何Unix / Posix文件系统
- RAID支持的文件系统
- 联网的分布式文件系统（NFS，HDFS等）
- IPFS
- 资料库
- NAS系统
- 原始串行或块设备
- 原始硬盘驱动器（hdd扇区等）

实现应实现对主机OS文件系统的支持。实现可以实现对其他存储系统的支持。

### 2.2.2[文件币片](https://spec.filecoin.io/#section-systems.filecoin_files.piece)

该*Filecoin片*是主要的*谈判单位*为用户存储Filecoin网络上的数据。Filecoin Piece*不是存储单位*，它没有特定大小，但是受*Sector*大小的限制。Filecoin片段的大小可以任意，但是如果片段大于矿工支持的扇区的大小，则必须将其拆分成更多的片段，以使每个片段都适合一个扇区。

A`Piece`是代表a的全部或一部分的对象，`File`由`Storage Clients`和`Storage Miners`在中使用`Deals`。`Storage Clients`租用`Storage Miners`存放`Pieces`。

Piece数据结构用于证明存储任意IPLD图和客户端数据。该图显示了一个部件及其证明树的详细组成，包括完整的和带宽优化的部件数据结构。

![image-20210201105729280](https://tva1.sinaimg.cn/large/008eGmZEgy1gn7ulf31hej31900pmn9l.jpg)

#### 2.2.2.1[数据表示](https://spec.filecoin.io/#section-systems.filecoin_files.piece.data-representation)

重要的是要强调，提交给Filecoin网络的数据在经过转换之后才变成`StorageProvider`存储数据的格式。

从用户开始准备要存储在Filecoin中的文件到提供者生成存储在一个部门中的所有作品标识符的过程，下面是过程。

前三个步骤在客户端进行。

1. 当客户想要在Filecoin网络中存储文件时，他们首先生成文件的IPLD DAG。表示DAG根节点的哈希是IPFS样式的CID，称为*有效负载CID*。
2. 为了制作*Filecoin Piece*，IPLD DAG被序列化为 [“ Content-Addressable aRchive”（。car）](https://github.com/ipld/specs/blob/master/block-layer/content-addressable-archives.md#summary)文件，该文件为原始字节格式。CAR文件是一个不透明的数据块，可打包在一起并传输IPLD节点。该*有效载荷CID*是CAR'ed和未CAR'ed结构之间常见。当稍后在存储客户端和存储提供者之间传输数据时，这将在稍后的数据检索期间有所帮助。
3. 生成的.car文件用额外的零位*填充*，以使该文件形成二进制Merkle树。为了获得干净的二进制Merkle树，.car文件的大小必须为2（^ 2）的幂。填充过程称为`Fr32 padding`，将每256位中的254位中的两（2）个零位加到输入文件中。下一步，填充过程获取过程的输出，`Fr32 padding`并找到其上方的大小，从而得到2的幂。`Fr32 padding`下一个2的幂次方的结果之间的差距用零填充。

为了说明这些步骤背后的原因，重要的是了解`StorageClient`和之间的总体谈判过程`StorageProvider`。CID或CommP是客户与存储提供商协商并同意的交易中包含的内容。达成协议后，客户端将文件发送给提供者（使用GraphSync）。提供者必须从接收到的文件中构造出CAR文件，并从其一侧导出Piece CID。为了避免客户向商定的文件发送不同的文件，提供者生成的件CID必须与先前协商的交易中包含的文件CID相同。

以下步骤在`StorageProvider`一侧进行（除了步骤4，也可以在客户端进行）。

1. 一旦`StorageProvider`接收到来自客户端的文件，他们就会从Piece的哈希值（填充的.car文件）中计算出Merkle根。干净的二叉树Merkle树的结果根是**Piece CID**。这也称为*CommP*或*计件承诺*，如前所述，必须与交易中包含的*承诺*相同。
2. 该部分与其他交易的数据一起包含在一个部门中。在`StorageProvider`随后计算所有部门内件梅克尔根。该树的根是*CommD*（又称*数据承诺*或`UnsealedSectorCID`）。
3. 所述`StorageProvider`然后密封该扇区并且将所得梅克尔根的根是*CommRLast*。
4. 特别是复制证明（PoRep），特别是SDR，会生成另一个名为*CommC的*Merkle根哈希，以证明已正确执行了承诺为*CommD*的数据的复制。
5. 最后，*CommR*（或*复制承诺*）是CommC的哈希。CommRLast。

**重要笔记：**

- `Fr32`是字段元素的32位表示形式（在我们的情况下为BLS12-381的算术字段）。为了格式正确，类型值`Fr32`必须*实际上*适合该字段，但是类型系统不强制执行该值。它是不变量，必须正确使用才能保留。在所谓的情况下`Fr32 padding`，两个零位被插入到一个“需要”最多254位才能表示的数字之后。这保证了结果将为`Fr32`，而不管初始254位的值如何。这是一种“保守”技术，因为对于某些初始值，实际上只需要一点零填充。
- 上面的步骤2和3是特定于Lotus实现的。可以通过不同的方式来实现相同的结果，例如，无需使用`Fr32`位填充。但是，任何实现都必须确保对初始IPLD DAG进行了序列化和填充，以便提供干净的二叉树，因此，从所得数据斑点中计算出Merkle根时，将得到相同的**Piece CID**。只要是这种情况，实现可能会偏离上面的前三个步骤。
- 最后，重要的是添加与*有效载荷CID*（在上面的前两个步骤中讨论过）和数据检索过程有关的注释。检索交易根据*有效负载CID*进行协商。达成检索协议后，检索矿工将开始将未密封和“未CAR'ed”的文件发送给客户端。传输从IPLD Merkle树的根节点开始，这样客户端可以从传输开始就验证*有效负载CID*，并验证他们接收的文件是他们在交易中协商的文件，而不是随机位。

#### 2.2.2.2[件商店](https://spec.filecoin.io/#section-systems.filecoin_files.piece.piecestore)

该`PieceStore`模块允许从本地存储中存储和检索碎片。零配件商店的主要目标是帮助 [存储](https://github.com/filecoin-project/go-fil-markets/blob/master/storagemarket)和 [检索市场](https://github.com/filecoin-project/go-fil-markets/blob/master/retrievalmarket)模块查找密封数据在部门内部的位置。存储市场写入数据，而检索市场读取数据，以便发送给检索客户。

在[此处](https://github.com/filecoin-project/go-fil-markets/tree/master/piecestore)可以找到PieceStore模块的实现 。

### 2.2.3[Filecoin中的数据传输](https://spec.filecoin.io/#section-systems.filecoin_files.data_transfer)

的*数据传输协议*是用于传输的全部或一部分的协议`Piece`跨越网络时处理是制成。数据传输模块的总体目标是使其成为底层传输介质的抽象，通过该底层传输介质在Filecoin网络中不同各方之间传输数据。当前，用于实际进行数据传输的底层介质或协议是GraphSync。这样，可以将数据传输协议视为协商协议。

数据传输协议既用于存储又用于检索交易。在这两种情况下，数据传输请求都是由客户端发起的。这样做的主要原因是，客户端通常比NAT落后很多，因此从其端开始任何数据传输更加方便。对于存储交易，数据传输请求将作为*推送请求*启动，以将数据发送到存储提供商。在“检索交易”的情况下，数据传输请求作为*拉取请求*启动，以由存储提供商检索数据。

发起数据传输的请求包括凭证或令牌（不要与付款渠道凭证混淆），该凭证或令牌指向双方之前已达成的特定交易。这样一来，存储提供商便可以识别请求并将其链接到已同意的交易，而不会忽略该请求。如下所述，检索交易的情况可能略有不同，在此情况下，交易提议和数据传输请求都可以一次发送。

#### 2.2.3.1[模组](https://spec.filecoin.io/#section-systems.filecoin_files.data_transfer.modules)

该图显示了数据传输及其模块如何与存储和检索市场相匹配。特别要注意，如何将来自市场的数据传输请求验证器插入“数据传输”模块，但其代码属于市场系统。

![image-20210201105943338](https://tva1.sinaimg.cn/large/008eGmZEgy1gn7unqn5ipj31980pan1w.jpg)

#### 2.2.3.2[术语](https://spec.filecoin.io/#section-systems.filecoin_files.data_transfer.terminology)

- **推送请求**：将数据发送给另一方的请求-通常由客户端发起，主要是在发生存储交易的情况下。
- 提取**请求**：请求对方发送数据的请求-通常由客户发起，主要是在“检索交易”的情况下。
- **请求者**：发起数据传输请求的一方（无论是推还是拉）-通常至少在Filecoin中当前实现的客户端，以克服NAT遍历问题。
- **响应者**：接收数据传输请求的一方-通常是存储提供者。
- **数据传输凭证或令牌**：围绕存储或检索相关数据的包装，可以识别和验证向另一方的传输请求。
- **请求验证器**：仅当响应者可以验证请求是否直接绑定到现有存储或检索交易时，数据传输模块才启动传输。验证不由数据传输模块本身执行。取而代之的是，请求验证器检查数据传输凭单以确定是否响应请求或不理会请求。
- **运输者**：协商和确认请求后，实际的转移由双方的运输者管理。传输器是数据传输模块的一部分，但与协商过程隔离。它可以访问基础可验证的传输协议，并使用它来发送数据和跟踪进度。
- **订户**：一个外部组件，通过订阅数据传输事件（例如进度或完成）来监视数据传输的进度。
- **GraphSync**：传输程序使用的默认基础传输协议。完整的graphsync规范可在[此处](https://github.com/ipld/specs/blob/master/block-layer/graphsync/graphsync.md)找到

#### 2.2.3.3[请求阶段](https://spec.filecoin.io/#section-systems.filecoin_files.data_transfer.request-phases)

任何数据传输都有两个基本阶段：

1. 协商：请求者和响应者通过使用数据传输凭证验证传输来同意传输。
2. 传输：协商阶段完成后，实际上就传输了数据。用于进行传输的默认协议是Graphsync。

请注意，“协商”和“转移”阶段可以发生在单独的往返行程中，也可能在相同的往返行程中发生，其中请求方通过发送请求隐式地同意，而响应方可以同意并立即发送或接收数据。该过程是在一次还是多次往返中进行，部分取决于请求是推式请求（存储交易）还是拉取请求（检索交易），以及数据传输协商过程是否能够执行回到底层的运输机制上。在使用GraphSync作为传输机制的情况下，可以使用[GraphSync的内置可扩展性](https://github.com/ipld/specs/blob/master/block-layer/graphsync/graphsync.md#extensions)将数据传输请求作为对GraphSync协议 [的扩展](https://github.com/ipld/specs/blob/master/block-layer/graphsync/graphsync.md#extensions)。因此，拉取请求仅需要一次往返。但是，由于Graphsync是不直接支持`push`类型请求的请求/响应协议，因此在Push情况下，协商是通过数据传输自身的libp2p协议在单独的请求中进行的`/fil/datatransfer/1.0.0`。其他未来的运输机制可能会同时处理“推”和“推”，也可能不会一次处理。接收到数据传输请求后，数据传输模块会对凭证进行解码，并将其交付给请求验证器。在存储交易中，请求验证程序检查所包含的交易是否是收件人之前已经同意的交易。对于检索交易，请求包括有关检索交易本身的建议。只要请求验证者接受交易建议，所有操作都将作为一次往返立即完成。

值得注意的是，在取回的情况下，提供者可以接受交易和数据传输请求，但是可以暂停取回本身以执行启封过程。在开始实际的数据传输之前，存储提供商必须解封所有请求的数据。此外，存储提供商可以选择在开始开封过程之前暂停检索流程，以请求开封付款请求。存储供应商可以选择要求支付这笔款项，以支付不可思议的计算成本，并避免成为行为不端的客户的受害者。

#### 2.2.3.4 流程示例

##### 2.2.3.4.1 推流

![image-20210201110343745](https://tva1.sinaimg.cn/large/008eGmZEgy1gn7urnqsigj31cs0owjuk.jpg)

1. 当请求者想要将数据发送给另一方时，它会发起“推”传输。
2. 请求者的数据传输模块将把推送请求与数据传输凭证一起发送给响应者。
3. 响应者的数据传输模块通过验证器验证数据传输请求，该验证器是响应者提供的依赖项。
4. 响应者的数据传输模块通过发出GraphSync请求来启动传输。
5. 请求者接收GraphSync请求，验证它是否识别出数据传输并开始发送数据。
6. 响应者接收数据并可以产生进度指示。
7. 响应者完成接收数据，并通知所有侦听器。

推送流程非常适合存储交易，在存储交易中，一旦提供者表明他们打算接受并发布客户的交易建议，客户便会立即启动数据传输。

#### 2.2.3.5 拉流-单程往返

![image-20210201110446569](https://tva1.sinaimg.cn/large/008eGmZEgy1gn7usyj977j31a40pmtbe.jpg)

#### 2.2.3.6[协议](https://spec.filecoin.io/#section-systems.filecoin_files.data_transfer.protocol)

可以通过数据传输协议（libp2p协议类型）在网络上协商数据传输。

使用数据传输协议作为独立的libp2p通讯机制并不是硬性要求-只要双方都实现了可以与对方通信的数据传输子系统，任何传输机制（包括离线机制）都是可以接受的。

#### 2.2.3.7[ 数据结构](https://spec.filecoin.io/#section-systems.filecoin_files.data_transfer.data-structures)

[示例：数据传输类型](https://spec.filecoin.io/#example-data-transfer-types)

```go
package datatransfer

import (
	"fmt"

	"github.com/ipfs/go-cid"
	"github.com/ipld/go-ipld-prime"
	"github.com/libp2p/go-libp2p-core/peer"

	"github.com/filecoin-project/go-data-transfer/encoding"
)

//go:generate cbor-gen-for ChannelID

// TypeIdentifier is a unique string identifier for a type of encodable object in a
// registry
type TypeIdentifier string

// EmptyTypeIdentifier means there is no voucher present
const EmptyTypeIdentifier = TypeIdentifier("")

// Registerable is a type of object in a registry. It must be encodable and must
// have a single method that uniquely identifies its type
type Registerable interface {
	encoding.Encodable
	// Type is a unique string identifier for this voucher type
	Type() TypeIdentifier
}

// Voucher is used to validate
// a data transfer request against the underlying storage or retrieval deal
// that precipitated it. The only requirement is a voucher can read and write
// from bytes, and has a string identifier type
type Voucher Registerable

// VoucherResult is used to provide option additional information about a
// voucher being rejected or accepted
type VoucherResult Registerable

// TransferID is an identifier for a data transfer, shared between
// request/responder and unique to the requester
type TransferID uint64

// ChannelID is a unique identifier for a channel, distinct by both the other
// party's peer ID + the transfer ID
type ChannelID struct {
	Initiator peer.ID
	Responder peer.ID
	ID        TransferID
}

func (c ChannelID) String() string {
	return fmt.Sprintf("%s-%s-%d", c.Initiator, c.Responder, c.ID)
}

// OtherParty returns the peer on the other side of the request, depending
// on whether this peer is the initiator or responder
func (c ChannelID) OtherParty(thisPeer peer.ID) peer.ID {
	if thisPeer == c.Initiator {
		return c.Responder
	}
	return c.Initiator
}

// Channel represents all the parameters for a single data transfer
type Channel interface {
	// TransferID returns the transfer id for this channel
	TransferID() TransferID

	// BaseCID returns the CID that is at the root of this data transfer
	BaseCID() cid.Cid

	// Selector returns the IPLD selector for this data transfer (represented as
	// an IPLD node)
	Selector() ipld.Node

	// Voucher returns the voucher for this data transfer
	Voucher() Voucher

	// Sender returns the peer id for the node that is sending data
	Sender() peer.ID

	// Recipient returns the peer id for the node that is receiving data
	Recipient() peer.ID

	// TotalSize returns the total size for the data being transferred
	TotalSize() uint64

	// IsPull returns whether this is a pull request
	IsPull() bool

	// ChannelID returns the ChannelID for this request
	ChannelID() ChannelID

	// OtherPeer returns the counter party peer for this channel
	OtherPeer() peer.ID
}

// ChannelState is channel parameters plus it's current state
type ChannelState interface {
	Channel

	// SelfPeer returns the peer this channel belongs to
	SelfPeer() peer.ID

	// Status is the current status of this channel
	Status() Status

	// Sent returns the number of bytes sent
	Sent() uint64

	// Received returns the number of bytes received
	Received() uint64

	// Message offers additional information about the current status
	Message() string

	// Vouchers returns all vouchers sent on this channel
	Vouchers() []Voucher

	// VoucherResults are results of vouchers sent on the channel
	VoucherResults() []VoucherResult

	// LastVoucher returns the last voucher sent on the channel
	LastVoucher() Voucher

	// LastVoucherResult returns the last voucher result sent on the channel
	LastVoucherResult() VoucherResult

	// ReceivedCids returns the cids received so far on the channel
	ReceivedCids() []cid.Cid

	// Queued returns the number of bytes read from the node and queued for sending
	Queued() uint64
}
```

[示例：数据传输状态](https://spec.filecoin.io/#example-data-transfer-statuses)

```go
package datatransfer

// Status is the status of transfer for a given channel
type Status uint64

const (
	// Requested means a data transfer was requested by has not yet been approved
	Requested Status = iota

	// Ongoing means the data transfer is in progress
	Ongoing

	// TransferFinished indicates the initiator is done sending/receiving
	// data but is awaiting confirmation from the responder
	TransferFinished

	// ResponderCompleted indicates the initiator received a message from the
	// responder that it's completed
	ResponderCompleted

	// Finalizing means the responder is awaiting a final message from the initator to
	// consider the transfer done
	Finalizing

	// Completing just means we have some final cleanup for a completed request
	Completing

	// Completed means the data transfer is completed successfully
	Completed

	// Failing just means we have some final cleanup for a failed request
	Failing

	// Failed means the data transfer failed
	Failed

	// Cancelling just means we have some final cleanup for a cancelled request
	Cancelling

	// Cancelled means the data transfer ended prematurely
	Cancelled

	// InitiatorPaused means the data sender has paused the channel (only the sender can unpause this)
	InitiatorPaused

	// ResponderPaused means the data receiver has paused the channel (only the receiver can unpause this)
	ResponderPaused

	// BothPaused means both sender and receiver have paused the channel seperately (both must unpause)
	BothPaused

	// ResponderFinalizing is a unique state where the responder is awaiting a final voucher
	ResponderFinalizing

	// ResponderFinalizingTransferFinished is a unique state where the responder is awaiting a final voucher
	// and we have received all data
	ResponderFinalizingTransferFinished

	// ChannelNotFoundError means the searched for data transfer does not exist
	ChannelNotFoundError
)

// Statuses are human readable names for data transfer states
var Statuses = map[Status]string{
	// Requested means a data transfer was requested by has not yet been approved
	Requested:                           "Requested",
	Ongoing:                             "Ongoing",
	TransferFinished:                    "TransferFinished",
	ResponderCompleted:                  "ResponderCompleted",
	Finalizing:                          "Finalizing",
	Completing:                          "Completing",
	Completed:                           "Completed",
	Failing:                             "Failing",
	Failed:                              "Failed",
	Cancelling:                          "Cancelling",
	Cancelled:                           "Cancelled",
	InitiatorPaused:                     "InitiatorPaused",
	ResponderPaused:                     "ResponderPaused",
	BothPaused:                          "BothPaused",
	ResponderFinalizing:                 "ResponderFinalizing",
	ResponderFinalizingTransferFinished: "ResponderFinalizingTransferFinished",
	ChannelNotFoundError:                "ChannelNotFoundError",
}
```

[示例：数据传输管理器](https://spec.filecoin.io/#example-data-transfer-manager)

管理器是数据传输子系统的所有实现所呈现的核心接口

```go
type Manager interface {

	// Start initializes data transfer processing
	Start(ctx context.Context) error

	// OnReady registers a listener for when the data transfer comes on line
	OnReady(ReadyFunc)

	// Stop terminates all data transfers and ends processing
	Stop(ctx context.Context) error

	// RegisterVoucherType registers a validator for the given voucher type
	// will error if voucher type does not implement voucher
	// or if there is a voucher type registered with an identical identifier
	RegisterVoucherType(voucherType Voucher, validator RequestValidator) error

	// RegisterRevalidator registers a revalidator for the given voucher type
	// Note: this is the voucher type used to revalidate. It can share a name
	// with the initial validator type and CAN be the same type, or a different type.
	// The revalidator can simply be the sampe as the original request validator,
	// or a different validator that satisfies the revalidator interface.
	RegisterRevalidator(voucherType Voucher, revalidator Revalidator) error

	// RegisterVoucherResultType allows deserialization of a voucher result,
	// so that a listener can read the metadata
	RegisterVoucherResultType(resultType VoucherResult) error

	// RegisterTransportConfigurer registers the given transport configurer to be run on requests with the given voucher
	// type
	RegisterTransportConfigurer(voucherType Voucher, configurer TransportConfigurer) error

	// open a data transfer that will send data to the recipient peer and
	// transfer parts of the piece that match the selector
	OpenPushDataChannel(ctx context.Context, to peer.ID, voucher Voucher, baseCid cid.Cid, selector ipld.Node) (ChannelID, error)

	// open a data transfer that will request data from the sending peer and
	// transfer parts of the piece that match the selector
	OpenPullDataChannel(ctx context.Context, to peer.ID, voucher Voucher, baseCid cid.Cid, selector ipld.Node) (ChannelID, error)

	// send an intermediate voucher as needed when the receiver sends a request for revalidation
	SendVoucher(ctx context.Context, chid ChannelID, voucher Voucher) error

	// close an open channel (effectively a cancel)
	CloseDataTransferChannel(ctx context.Context, chid ChannelID) error

	// pause a data transfer channel (only allowed if transport supports it)
	PauseDataTransferChannel(ctx context.Context, chid ChannelID) error

	// resume a data transfer channel (only allowed if transport supports it)
	ResumeDataTransferChannel(ctx context.Context, chid ChannelID) error

	// get status of a transfer
	TransferChannelStatus(ctx context.Context, x ChannelID) Status

	// get notified when certain types of events happen
	SubscribeToEvents(subscriber Subscriber) Unsubscribe

	// get all in progress transfers
	InProgressChannels(ctx context.Context) (map[ChannelID]ChannelState, error)

	// RestartDataTransferChannel restarts an existing data transfer channel
	RestartDataTransferChannel(ctx context.Context, chid ChannelID) error
}
```

### 2.2.4[数据格式和序列化](https://spec.filecoin.io/#section-systems.filecoin_files.serialization)

Filecoin试图利用所需的数据格式更少，并采用规范化的序列化规则，以通过简单性提高协议安全性，并实现Filecoin协议实现之间的互操作性。

在[此处了解有关CBOR使用情况](https://github.com/filecoin-project/specs/issues/621)和 [Filecoin中的](https://github.com/filecoin-project/specs/issues/621)[int类型的](https://github.com/filecoin-project/specs/issues/615)更多设计注意 [事项](https://github.com/filecoin-project/specs/issues/615)。

#### 2.2.4.1[资料格式](https://spec.filecoin.io/#section-systems.filecoin_files.serialization.data-formats)

Filecoin内存中的数据类型通常很简单。实现应支持两种整数类型：Int（表示本机64位整数）和BigInt（表示任意长度），并避免处理浮点数以最大程度地减少跨编程语言和实现的互操作性问题。

您还可以在Filecoin协议中阅读有关[数据格式的](https://spec.filecoin.io/#section-algorithms.crypto.randomness)更多 [信息，作为随机性生成](https://spec.filecoin.io/#section-algorithms.crypto.randomness)的一部分。

#### 2.2.4.2[序列化](https://spec.filecoin.io/#section-systems.filecoin_files.serialization.serialization)

`Serialization`Filecoin中的数据可确保用于序列化内存中数据的一致格式，以进行传输和存储中传输。序列化对于Filecoin协议实现之间的协议安全性和互操作性至关重要，从而可以跨Filecoin节点进行一致的状态更新。

Filecoin中的所有数据结构都是 [CBOR](https://tools.ietf.org/html/rfc7049)元组编码的。也就是说，在Filecoin系统中使用的任何数据结构（本规范中的结构）都应按声明顺序序列化为CBOR数组，并带有与数据结构字段相对应的项。

你可以找到在CBOR主要数据类型的编码结构 [在这里](https://tools.ietf.org/html/rfc7049#section-2.1)。

为了说明起见，内存映射将以按预定顺序列出的键和值的CBOR数组表示。序列化格式的近期更新将涉及适当地标记字段，以确保随着协议的发展而进行适当的序列化/反序列化。

## 2.3[虚拟机](https://spec.filecoin.io/#section-systems.filecoin_vm)

Filecoin区块链中的Actor等同于以太坊虚拟机中的智能合约。

Filecoin虚拟机（VM）是负责执行所有参与者代码的系统组件。在Filecoin VM上执行参与者（即链上执行）会产生汽油费用。

在Filecoin VM上应用（即执行）的任何操作都将以*状态树*的形式产生输出（如下所述）。最新的*状态树*是Filecoin区块链中当前的真相来源。该*国树*是由CID，其存储在IPLD店鉴定。

### 2.3.1[VM Actor接口](https://spec.filecoin.io/#section-systems.filecoin_vm.actor)

如上所述，Actor是以太坊虚拟机中智能合约的Filecoin等效项。因此，Actor是系统的核心组件。Filecoin区块链当前状态的任何更改都必须通过参与者方法调用来触发。

本小节描述Actor与Filecoin虚拟机之间的*接口*。这意味着下面描述的大多数内容并不严格属于VM。相反，逻辑位于VM和Actors逻辑之间的接口上。

总共共有十一（11）种类型的*内置*Actor，但并非所有类型都与VM交互。一些Actor不会调用对区块链的StateTree的更改，因此不需要与VM的接口。我们稍后将在“系统参与者”小节中讨论所有系统参与者的详细信息。

该*演员地址*是通过散列发送者的公钥和创建随机数生成稳定的地址。在整个链重组中应该保持稳定。该*演员ID地址*，另一方面，是紧凑的，但可以在链重新组织的情况下更改自动递增地址。话虽如此，演员创建后应该使用*演员地址*。

[例：](https://spec.filecoin.io/#example-)

```go
package builtin

import (
	addr "github.com/filecoin-project/go-address"
)

// Addresses for singleton system actors.
var (
	// Distinguished AccountActor that is the source of system implicit messages.
	SystemActorAddr           = mustMakeAddress(0)
	InitActorAddr             = mustMakeAddress(1)
	RewardActorAddr           = mustMakeAddress(2)
	CronActorAddr             = mustMakeAddress(3)
	StoragePowerActorAddr     = mustMakeAddress(4)
	StorageMarketActorAddr    = mustMakeAddress(5)
	VerifiedRegistryActorAddr = mustMakeAddress(6)
	// Distinguished AccountActor that is the destination of all burnt funds.
	BurntFundsActorAddr = mustMakeAddress(99)
)

const FirstNonSingletonActorId = 100

func mustMakeAddress(id uint64) addr.Address {
	address, err := addr.NewIDAddress(id)
	if err != nil {
		panic(err)
	}
	return address
}
```

该`ActorState`结构由参与者的余额（根据该参与者持有的令牌）以及一组用于查询，检查链状态并与之交互的状态方法组成。

### 2.3.2[状态树](https://spec.filecoin.io/#section-systems.filecoin_vm.state_tree)

状态树是对Filecoin区块链应用的任何操作的执行输出。链上（即VM）状态数据结构是将地址绑定到参与者状态的映射（以散列阵列映射Trie-HAMT的形式）。VM在每次执行actor方法时都会调用当前的State Tree函数。

[示例：StateTree](https://spec.filecoin.io/#example-statetree)

StateTree存储参与者的ID。

```go
type StateTree struct {
	root        adt.Map
	version     types.StateTreeVersion
	info        cid.Cid
	Store       cbor.IpldStore
	lookupIDFun func(address.Address) (address.Address, error)

	snaps *stateSnaps
}
```

### 2.3.3[VM消息-Actor方法调用](https://spec.filecoin.io/#section-systems.filecoin_vm.message)

消息是两个参与者之间进行通信的单位，因此是状态变化的根本原因。一条消息结合了：

- 从发送方转移到接收方的令牌金额，以及
- 具有在接收方上调用的参数的方法（可选/在适用的情况下）。

演员代码可以在处理收到的消息时向其他演员发送其他消息。消息是同步处理的，也就是说，参与者在恢复控制之前等待发送的消息完成。

消息的处理消耗了计算和存储单位，两者均以gas表示。消息的*气体限制*为处理该消息提供了所需的计算上限。消息的发件人以其确定的汽油价格来支付消息执行所消耗的气体单位（包括所有嵌套的消息）。区块生产者选择要包含在区块中的消息，并根据每个消息的汽油价格和消耗量获得奖励，从而形成市场。

#### 2.3.3.1[消息语法验证](https://spec.filecoin.io/#section-systems.filecoin_vm.message.message-syntax-validation)

语法无效的消息不得传输，保留在消息池中或包含在块中。如果收到无效消息，则应将其丢弃，并且不要进一步传播。

当单独发送时（在包含在块中之前）`SignedMessage`，无论使用哪种签名方案，都将消息打包为 。有效的签名邮件的序列化总大小不大于`message.MessageMaxSize`。

```go
type SignedMessage struct {
	Message   Message
	Signature crypto.Signature
}
```

语法上有效的`UnsignedMessage`：

- 具有格式正确的非空`To`地址，
- 具有格式正确的非空`From`地址，
- 具有`Value`不小于零且不大于令牌总供给（`2e9 * 1e18`），并且
- 具有非负数`GasPrice`，
- 具有`GasLimit`至少等于与消息的序列化字节关联的气体消耗的值，
- 具有`GasLimit`不大于区块气体限制网络参数的值。

```go
type Message struct {
	// Version of this message (has to be non-negative)
	Version uint64

	// Address of the receiving actor.
	To   address.Address
	// Address of the sending actor.
	From address.Address

	CallSeqNum uint64

	// Value to transfer from sender's to receiver's balance.
	Value BigInt

	// GasPrice is a Gas-to-FIL cost
	GasPrice BigInt
	// Maximum Gas to be spent on the processing of this message
	GasLimit int64

	// Optional method to invoke on receiver, zero for a plain value transfer.
	Method abi.MethodNum
	//Serialized parameters to the method.
	Params []byte
}
```

应该有几个功能可以从中提取信息`Message struct`，例如发件人和收件人地址，要转移的值，执行消息所需的资金以及消息的CID。

假定消息最终应包含在一个块中并添加到区块链中，则应检查消息的发送者和接收者的消息有效性，该值（应为非负值，并且始终小于循环供应），天然气价格（该价格又应为非负数）且`BlockGasLimit`该价格不应大于该区块的天然气限额。

#### 2.3.3.2[消息语义验证](https://spec.filecoin.io/#section-systems.filecoin_vm.message.message-semantic-validation)

语义验证是指需要消息本身之外的信息的验证。

语义上有效的`SignedMessage`必须带有签名，该签名可验证有效载荷是否已被`From`地址标识的帐户执行者的公钥签名。请注意，当`From`地址是ID地址时，必须在块所标识的父状态下的发送帐户参与者的状态下查找公钥。

注意：发送方必须*以*包含消息*的块*所*标识的父级状态*存在。这意味着单个块包含创建新帐户actor的消息和来自同一actor的消息是无效的。来自该参与者的第一条消息必须等到下一个纪元。消息池可能会排除来自参与者的，尚未处于链状状态的消息。

消息没有进一步的语义验证，可能导致包含该消息的块无效。每个语法有效且正确签名的消息都可以包含在一个块中，并会从执行中产生一个收据。其中`MessageReceipt sturct`包括以下内容：

```go
type MessageReceipt struct {
	ExitCode exitcode.ExitCode
	Return   []byte
	GasUsed  int64
}
```

但是，消息可能无法执行到完成，在这种情况下，它不会触发所需的状态更改。

这种“无消息语义验证”策略的原因是，在消息*作为提示集的一部分*执行之前，将不知道消息将应用于的状态。块生产者不知道在提示集中是否有另一个块会在它之前，因此从声明的父状态更改了该块消息将应用到的状态。

[例：](https://spec.filecoin.io/#example-)

```go
package types

import (
	"bytes"
	"encoding/json"
	"fmt"

	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/lotus/build"
	block "github.com/ipfs/go-block-format"
	"github.com/ipfs/go-cid"
	xerrors "golang.org/x/xerrors"

	"github.com/filecoin-project/go-address"
)

const MessageVersion = 0

type ChainMsg interface {
	Cid() cid.Cid
	VMMessage() *Message
	ToStorageBlock() (block.Block, error)
	// FIXME: This is the *message* length, this name is misleading.
	ChainLength() int
}

type Message struct {
	Version uint64

	To   address.Address
	From address.Address

	Nonce uint64

	Value abi.TokenAmount

	GasLimit   int64
	GasFeeCap  abi.TokenAmount
	GasPremium abi.TokenAmount

	Method abi.MethodNum
	Params []byte
}

func (m *Message) Caller() address.Address {
	return m.From
}

func (m *Message) Receiver() address.Address {
	return m.To
}

func (m *Message) ValueReceived() abi.TokenAmount {
	return m.Value
}

func DecodeMessage(b []byte) (*Message, error) {
	var msg Message
	if err := msg.UnmarshalCBOR(bytes.NewReader(b)); err != nil {
		return nil, err
	}

	if msg.Version != MessageVersion {
		return nil, fmt.Errorf("decoded message had incorrect version (%d)", msg.Version)
	}

	return &msg, nil
}

func (m *Message) Serialize() ([]byte, error) {
	buf := new(bytes.Buffer)
	if err := m.MarshalCBOR(buf); err != nil {
		return nil, err
	}
	return buf.Bytes(), nil
}

func (m *Message) ChainLength() int {
	ser, err := m.Serialize()
	if err != nil {
		panic(err)
	}
	return len(ser)
}

func (m *Message) ToStorageBlock() (block.Block, error) {
	data, err := m.Serialize()
	if err != nil {
		return nil, err
	}

	c, err := abi.CidBuilder.Sum(data)
	if err != nil {
		return nil, err
	}

	return block.NewBlockWithCid(data, c)
}

func (m *Message) Cid() cid.Cid {
	b, err := m.ToStorageBlock()
	if err != nil {
		panic(fmt.Sprintf("failed to marshal message: %s", err)) // I think this is maybe sketchy, what happens if we try to serialize a message with an undefined address in it?
	}

	return b.Cid()
}

type mCid struct {
	*RawMessage
	CID cid.Cid
}

type RawMessage Message

func (m *Message) MarshalJSON() ([]byte, error) {
	return json.Marshal(&mCid{
		RawMessage: (*RawMessage)(m),
		CID:        m.Cid(),
	})
}

func (m *Message) RequiredFunds() BigInt {
	return BigMul(m.GasFeeCap, NewInt(uint64(m.GasLimit)))
}

func (m *Message) VMMessage() *Message {
	return m
}

func (m *Message) Equals(o *Message) bool {
	return m.Cid() == o.Cid()
}

func (m *Message) EqualCall(o *Message) bool {
	m1 := *m
	m2 := *o

	m1.GasLimit, m2.GasLimit = 0, 0
	m1.GasFeeCap, m2.GasFeeCap = big.Zero(), big.Zero()
	m1.GasPremium, m2.GasPremium = big.Zero(), big.Zero()

	return (&m1).Equals(&m2)
}

func (m *Message) ValidForBlockInclusion(minGas int64) error {
	if m.Version != 0 {
		return xerrors.New("'Version' unsupported")
	}

	if m.To == address.Undef {
		return xerrors.New("'To' address cannot be empty")
	}

	if m.From == address.Undef {
		return xerrors.New("'From' address cannot be empty")
	}

	if m.Value.Int == nil {
		return xerrors.New("'Value' cannot be nil")
	}

	if m.Value.LessThan(big.Zero()) {
		return xerrors.New("'Value' field cannot be negative")
	}

	if m.Value.GreaterThan(TotalFilecoinInt) {
		return xerrors.New("'Value' field cannot be greater than total filecoin supply")
	}

	if m.GasFeeCap.Int == nil {
		return xerrors.New("'GasFeeCap' cannot be nil")
	}

	if m.GasFeeCap.LessThan(big.Zero()) {
		return xerrors.New("'GasFeeCap' field cannot be negative")
	}

	if m.GasPremium.Int == nil {
		return xerrors.New("'GasPremium' cannot be nil")
	}

	if m.GasPremium.LessThan(big.Zero()) {
		return xerrors.New("'GasPremium' field cannot be negative")
	}

	if m.GasPremium.GreaterThan(m.GasFeeCap) {
		return xerrors.New("'GasFeeCap' less than 'GasPremium'")
	}

	if m.GasLimit > build.BlockGasLimit {
		return xerrors.New("'GasLimit' field cannot be greater than a block's gas limit")
	}

	// since prices might vary with time, this is technically semantic validation
	if m.GasLimit < minGas {
		return xerrors.Errorf("'GasLimit' field cannot be less than the cost of storing a message on chain %d < %d", m.GasLimit, minGas)
	}

	return nil
}

const TestGasLimit = 100e6
```

### 2.3.4[VM运行时环境（在VM内部）](https://spec.filecoin.io/#section-systems.filecoin_vm.runtime)

#### 2.3.4.1[收据](https://spec.filecoin.io/#section-systems.filecoin_vm.runtime.receipts)

甲`MessageReceipt`包含一个顶层消息执行的结果。每个语法有效且正确签名的消息都可以包含在一个块中，并会从执行中产生一个收据。

语法有效的收据具有：

- 一个非负`ExitCode`，
- `Return`仅当退出代码为零时，才为非空值；并且
- 非负数`GasUsed`。

```go
type MessageReceipt struct {
	ExitCode exitcode.ExitCode
	Return   []byte
	GasUsed  int64
}
```

#### 2.3.4.2[`vm/runtime` 演员界面](https://spec.filecoin.io/#section-systems.filecoin_vm.runtime.vmruntime-actors-interface)

演员接口的实现可以在[这里](https://github.com/filecoin-project/specs-actors/blob/master/actors/runtime/runtime.go)找到

#### 2.3.4.3[`vm/runtime` 虚拟机实施](https://spec.filecoin.io/#section-systems.filecoin_vm.runtime.vmruntime-vm-implementation)

Filecoin虚拟机运行时的Lotus实现可在[此处](https://github.com/filecoin-project/lotus/blob/master/chain/vm/runtime.go)找到

#### 2.3.4.4[退出码](https://spec.filecoin.io/#section-systems.filecoin_vm.runtime.exit-codes)

有一些由不同参与者共享的常见运行时退出代码。它们的定义可以在[这里](https://github.com/filecoin-project/go-state-types/blob/master/exitcode/common.go)找到 。

### 2.3.5[煤气费](https://spec.filecoin.io/#section-systems.filecoin_vm.gas_fee)

#### 2.3.5.1[概要](https://spec.filecoin.io/#section-systems.filecoin_vm.gas_fee.summary)

与许多区块链的传统情况一样，Gas是衡量链上消息操作要执行多少消耗的存储和/或计算资源的度量单位。在较高级别上，它的工作方式如下：消息发送者指定他们愿意支付的最高金额，以便消息被执行并包含在块中。这是根据总的天然气单位数（`GasLimit`）（通常期望高于实际`GasUsed`单位）和每单位天然气的价格（或费用`GasFeeCap`）来指定的。

传统上，`GasUsed * GasFeeCap`去生产矿工作为奖励。该产品的结果被视为消息包含的优先费用，也就是说，消息以降序排列，而消息最高的消息`GasUsed * GasFeeCap`被优先考虑，因为它们会向矿工返回更多的利润。

但是，已经观察到，`GasUsed * GasFee`出于一些原因，这种（支付）策略对于块生产矿工是有问题的。首先，一个生产区块的矿工可能免费包含一条非常昂贵的消息（就所需的链资源而言），在这种情况下，链本身需要承担成本。其次，消息发送者可以任意设置高价，但对于低成本消息（同样以链资源而言），这会导致DoS漏洞。

为了克服这种情况，Filecoin区块链定义了一个`BaseFee`，每个消息都会对其进行刻录。理由是，考虑到天然气是衡量链上资源消耗的一种手段，与将其奖励给矿工相比，将其燃烧是有意义的。这样，避免了来自矿工的费用操纵。它`BaseFee`是动态的，会根据网络拥塞情况自动调整。这一事实使网络可以抵御垃圾邮件攻击。鉴于在SPAM攻击期间网络负载会增加，因此，攻击者无法将SPAM消息的完整块长时间保留`BaseFee`。

最后，`GasPremium`是发件人包括的优先费，以激励矿工选择最有利可图的消息。换句话说，如果消息发件人希望更快地包含其消息，则可以设置更高的`GasPremium`。

#### 2.3.5.2[参量](https://spec.filecoin.io/#section-systems.filecoin_vm.gas_fee.parameters)

- `GasUsed`是执行一条消息所消耗的资源（或气体单位）数量的度量。每种气体单位都以attoFIL进行测量，因此`GasUsed`是代表能耗单位的数字。`GasUsed`与消息是正确执行还是失败无关。
- `BaseFee`是每次执行消息时要燃烧（发送到不可恢复的地址）的每单位天然气的设定价格（以attoFIL /天然气单位计量）。的值`BaseFee`是动态的，并根据当前的网络拥塞参数进行调整。例如，当网络超出5B气体限制使用量时，`BaseFee`增加，而当气体限制使用量下降到5B以下时，情况相反。的`BaseFee`施加到每个块应被包括在块本身。应该有可能`BaseFee`从链的顶部获得电流值。的`BaseFee`每单位适用`GasUsed`，因此，气体的总量烧制的消息是`BaseFee * GasUsed`。请注意，`BaseFee`每条消息都会产生，但是同一块中所有消息的值都相同。
- `GasLimit`以气体为单位进行测量，并由消息发送者设置。它对应允许消息执行在链上消耗的气体量（即气体单位数）施加了硬性限制。消息触发的每个基本操作都会消耗气体，而消息用尽的消息将失败。当消息失败时，由于执行此消息而对状态进行的所有修改都将恢复为先前的状态。与消息执行是否成功无关，矿工将获得他们执行消息所消耗的资源的奖励（见`GasPremium`下文）。
- `GasFeeCap`是消息发件人愿意为每单位天然气支付的最高价格（以attoFIL / gas单位衡量）。再加上`GasLimit`，则`GasFeeCap`是设置一个发件人将一个消息FIL支付的最高金额：发件人是保证信息绝不会令他们超过`GasLimit * GasFeeCap`attoFIL（不包括任何溢价，该信息包含其收件人）。
- `GasPremium`是消息发送者愿意支付的每单位天然气价格（以attoFIL / gas计量）（在顶部`BaseFee`）以“提示”将包含该消息的矿工。一条消息通常可以`GasLimit * GasPremium`有效地使它的矿工attoFIL `GasPremium = GasFeeCap - BaseFee`。请注意，与相对于`GasPremium`应用于`GasLimit`，`GasUsed`以使矿工的消息选择更加直接。

[示例：ComputeGasOverestimationBurn](https://spec.filecoin.io/#example-computegasoverestimationburn)

ComputeGasOverestimationBurn计算要退款的燃气量和要燃烧的燃气量结果是（退款，燃烧）

```go
func ComputeGasOverestimationBurn(gasUsed, gasLimit int64) (int64, int64) {
	if gasUsed == 0 {
		return 0, gasLimit
	}

	// over = gasLimit/gasUsed - 1 - 0.1
	// over = min(over, 1)
	// gasToBurn = (gasLimit - gasUsed) * over

	// so to factor out division from `over`
	// over*gasUsed = min(gasLimit - (11*gasUsed)/10, gasUsed)
	// gasToBurn = ((gasLimit - gasUsed)*over*gasUsed) / gasUsed
	over := gasLimit - (gasOveruseNum*gasUsed)/gasOveruseDenom
	if over < 0 {
		return gasLimit - gasUsed, 0
	}

	// if we want sharper scaling it goes here:
	// over *= 2

	if over > gasUsed {
		over = gasUsed
	}

	// needs bigint, as it overflows in pathological case gasLimit > 2^32 gasUsed = gasLimit / 2
	gasToBurn := big.NewInt(gasLimit - gasUsed)
	gasToBurn = big.Mul(gasToBurn, big.NewInt(over))
	gasToBurn = big.Div(gasToBurn, big.NewInt(gasUsed))

	return gasLimit - gasUsed - gasToBurn.Int64(), gasToBurn.Int64()
}
```

[示例：ComputeNextBaseFee](https://spec.filecoin.io/#example-computenextbasefee)

```go
func ComputeNextBaseFee(baseFee types.BigInt, gasLimitUsed int64, noOfBlocks int, epoch abi.ChainEpoch) types.BigInt {
	// deta := gasLimitUsed/noOfBlocks - build.BlockGasTarget
	// change := baseFee * deta / BlockGasTarget
	// nextBaseFee = baseFee + change
	// nextBaseFee = max(nextBaseFee, build.MinimumBaseFee)

	var delta int64
	if epoch > build.UpgradeSmokeHeight {
		delta = gasLimitUsed / int64(noOfBlocks)
		delta -= build.BlockGasTarget
	} else {
		delta = build.PackingEfficiencyDenom * gasLimitUsed / (int64(noOfBlocks) * build.PackingEfficiencyNum)
		delta -= build.BlockGasTarget
	}

	// cap change at 12.5% (BaseFeeMaxChangeDenom) by capping delta
	if delta > build.BlockGasTarget {
		delta = build.BlockGasTarget
	}
	if delta < -build.BlockGasTarget {
		delta = -build.BlockGasTarget
	}

	change := big.Mul(baseFee, big.NewInt(delta))
	change = big.Div(change, big.NewInt(build.BlockGasTarget))
	change = big.Div(change, big.NewInt(build.BaseFeeMaxChangeDenom))

	nextBaseFee := big.Add(baseFee, change)
	if big.Cmp(nextBaseFee, big.NewInt(build.MinimumBaseFee)) < 0 {
		nextBaseFee = big.NewInt(build.MinimumBaseFee)
	}
	return nextBaseFee
}
```

#### 2.3.5.3[注释与含义](https://spec.filecoin.io/#section-systems.filecoin_vm.gas_fee.notes--implications)

- 的值`GasFeeCap`应始终高于网络的`BaseFee`。如果消息的`GasFeeCap`低于`BaseFee`，则其余部分来自矿工（作为罚款）。由于矿工选择的消息的价格低于`BaseFee`网络费用（即，不包括网络费用），因此对矿工施加此罚款。然而，矿工可能要选择一个消息，其`GasFeeCap`比小`BaseFee`，如果同一个发件人在邮件池，其另一条消息`GasFeeCap`比要大得多`BaseFee`。回想一下，如果存在多个矿工，则矿工应该从消息池中选择发件人的所有消息。理由是增加第二条消息的费用将弥补第一条消息的损失。
- 如果`BaseFee + GasPremium`> `GasFeeCap`，则矿工可能不会获得全部`GasLimit * GasPremium`作为奖励。
- 一条消息的花费不得超过`GasFeeCap * GasLimit`。从该金额中，网络首先`BaseFee`被支付（烧掉）。之后，最多`GasLimit * GasPremium`将给予矿工作为奖励。
- 耗尽气体的消息失败，并显示“耗尽气体”退出代码。`GasUsed * BaseFee`仍将被烧毁（在这种情况下`GasUsed = GasLimit`），而矿工仍将得到奖励`GasLimit * GasPremium`。这是假定`GasFeeCap > BaseFee + GasPremium`。
- 较低的价格`GasFeeCap`可能会导致消息滞留在消息池中，因为对于任何矿工来说，选择它并将其包含在一个块中，在利润方面都不够吸引人。发生这种情况时，将有一个更新程序，以`GasFeeCap`使消息对矿工更具吸引力。发送者可以将新消息推送到消息池（默认情况下，该消息池将传播到其他矿工的消息池），其中：i）旧消息和新消息的标识符相同（例如，相同`Nonce`），并且ii）`GasPremium`更新并增加至少25％的先前值。

### 2.3.6[系统角色](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors)

总共有十一（11）个内置的System Actor，但是并不是所有的Actor都与VM交互。每个演员都由*代码ID*（或CID）*标识*。

VM处理需要两个系统参与者：

- 在 [InitActor](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.initactor)，初始化新的参与者和记录网络名称，
- 在 [CronActor](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.cronactor)，，在每个时间段运行关键功能的调度演员。还有两个与VM交互的参与者：
- 负责用户帐户（非单一帐户）的 [AccountActor](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.accountactor)，以及
- 该 [RewardActor](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.rewardactor)块奖励和令牌归属（单）。

不直接与VM交互的其余七（7）个内置系统角色是：

- `StorageMarketActor`：负责管理存储和检索交易[ [Market Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/market/market_actor.go) ]
- `StorageMinerActor`：负责处理采矿业务并收集证据的[演员](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go)[ [Storage Miner Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go) ]
- `MultisigActor`（或Multi-Signature Wallet Actor）：负责处理涉及Filecoin钱包的操作[ [Multisig Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/multisig/multisig_actor.go) ]
- `PaymentChannelActor`：负责建立和结算与支付渠道有关的资金[ [Paych Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/paych/paych_actor.go) ]
- `StoragePowerActor`：负责跟踪每个存储矿工分配的存储功率[ [Storage Power Actor](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/power/power_actor.go) ]
- `VerifiedRegistryActor`：负责管理已验证的客户[ [Verifreg Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/verifreg/verified_registry_actor.go) ]
- `SystemActor`：普通系统演员[ [System Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/system/system_actor.go) ]

#### 2.3.6.1[ CronActor](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.cronactor)

内置在创世状态中，`CronActor`的分派表调用`StoragePowerActor`和`StorageMarketActor`，以维护内部状态并处理延迟的事件。在网络升级后，它原则上可以调用其他参与者。

[例：](https://spec.filecoin.io/#example-)

```go
package cron

import (
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/cbor"
	cron0 "github.com/filecoin-project/specs-actors/actors/builtin/cron"
	"github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v2/actors/builtin"
	"github.com/filecoin-project/specs-actors/v2/actors/runtime"
)

// The cron actor is a built-in singleton that sends messages to other registered actors at the end of each epoch.
type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.EpochTick,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.CronActorCodeID
}

func (a Actor) IsSingleton() bool {
	return true
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

//type ConstructorParams struct {
//	Entries []Entry
//}
type ConstructorParams = cron0.ConstructorParams

type EntryParam = cron0.Entry

func (a Actor) Constructor(rt runtime.Runtime, params *ConstructorParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)
	entries := make([]Entry, len(params.Entries))
	for i, e := range params.Entries {
		entries[i] = Entry(e) // Identical
	}
	rt.StateCreate(ConstructState(entries))
	return nil
}

// Invoked by the system after all other messages in the epoch have been processed.
func (a Actor) EpochTick(rt runtime.Runtime, _ *abi.EmptyValue) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)

	var st State
	rt.StateReadonly(&st)
	for _, entry := range st.Entries {
		_ = rt.Send(entry.Receiver, entry.MethodNum, nil, abi.NewTokenAmount(0), &builtin.Discard{})
		// Any error and return value are ignored.
	}

	return nil
}
```

#### 2.3.6.2[初始化演员](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.initactor)

将`InitActor`有可能创造新的角色，例如，那些进入系统的电源。它维护着一个表，用于将公共密钥和临时参与者地址解析为其规范的ID地址。无效的CID不应提交给状态树。

请注意，在进行链重组时，规范ID地址不会保留。演员地址或公钥在链重组后仍然有效。

[例：](https://spec.filecoin.io/#example-)

```go
package init

import (
	addr "github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/exitcode"
	init0 "github.com/filecoin-project/specs-actors/actors/builtin/init"
	cid "github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v2/actors/builtin"
	"github.com/filecoin-project/specs-actors/v2/actors/runtime"
	autil "github.com/filecoin-project/specs-actors/v2/actors/util"
	"github.com/filecoin-project/specs-actors/v2/actors/util/adt"
)

// The init actor uniquely has the power to create new actors.
// It maintains a table resolving pubkey and temporary actor addresses to the canonical ID-addresses.
type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.Exec,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.InitActorCodeID
}

func (a Actor) IsSingleton() bool {
	return true
}

func (a Actor) State() cbor.Er { return new(State) }

var _ runtime.VMActor = Actor{}

//type ConstructorParams struct {
//	NetworkName string
//}
type ConstructorParams = init0.ConstructorParams

func (a Actor) Constructor(rt runtime.Runtime, params *ConstructorParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)
	emptyMap, err := adt.MakeEmptyMap(adt.AsStore(rt)).Root()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to construct state")

	st := ConstructState(emptyMap, params.NetworkName)
	rt.StateCreate(st)
	return nil
}

//type ExecParams struct {
//	CodeCID           cid.Cid `checked:"true"` // invalid CIDs won't get committed to the state tree
//	ConstructorParams []byte
//}
type ExecParams = init0.ExecParams

//type ExecReturn struct {
//	IDAddress     addr.Address // The canonical ID-based address for the actor.
//	RobustAddress addr.Address // A more expensive but re-org-safe address for the newly created actor.
//}
type ExecReturn = init0.ExecReturn

func (a Actor) Exec(rt runtime.Runtime, params *ExecParams) *ExecReturn {
	rt.ValidateImmediateCallerAcceptAny()
	callerCodeCID, ok := rt.GetActorCodeCID(rt.Caller())
	autil.AssertMsg(ok, "no code for actor at %s", rt.Caller())
	if !canExec(callerCodeCID, params.CodeCID) {
		rt.Abortf(exitcode.ErrForbidden, "caller type %v cannot exec actor type %v", callerCodeCID, params.CodeCID)
	}

	// Compute a re-org-stable address.
	// This address exists for use by messages coming from outside the system, in order to
	// stably address the newly created actor even if a chain re-org causes it to end up with
	// a different ID.
	uniqueAddress := rt.NewActorAddress()

	// Allocate an ID for this actor.
	// Store mapping of pubkey or actor address to actor ID
	var st State
	var idAddr addr.Address
	rt.StateTransaction(&st, func() {
		var err error
		idAddr, err = st.MapAddressToNewID(adt.AsStore(rt), uniqueAddress)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to allocate ID address")
	})

	// Create an empty actor.
	rt.CreateActor(params.CodeCID, idAddr)

	// Invoke constructor.
	code := rt.Send(idAddr, builtin.MethodConstructor, builtin.CBORBytes(params.ConstructorParams), rt.ValueReceived(), &builtin.Discard{})
	builtin.RequireSuccess(rt, code, "constructor failed")

	return &ExecReturn{IDAddress: idAddr, RobustAddress: uniqueAddress}
}

func canExec(callerCodeID cid.Cid, execCodeID cid.Cid) bool {
	switch execCodeID {
	case builtin.StorageMinerActorCodeID:
		if callerCodeID == builtin.StoragePowerActorCodeID {
			return true
		}
		return false
	case builtin.PaymentChannelActorCodeID, builtin.MultisigActorCodeID:
		return true
	default:
		return false
	}
}
```

#### 2.3.6.3[奖励演员](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.rewardactor)

的`RewardActor`就是unminted Filecoin令牌将被保留。演员直接将奖励分配给矿工演员，他们被锁定以归属。当前纪元的奖励值在纪元末通过cron tick更新。

[例：](https://spec.filecoin.io/#example-)

```go
package reward

import (
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/exitcode"
	rtt "github.com/filecoin-project/go-state-types/rt"
	reward0 "github.com/filecoin-project/specs-actors/actors/builtin/reward"
	"github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v2/actors/builtin"
	"github.com/filecoin-project/specs-actors/v2/actors/runtime"
	. "github.com/filecoin-project/specs-actors/v2/actors/util"
	"github.com/filecoin-project/specs-actors/v2/actors/util/smoothing"
)

// PenaltyMultiplier is the factor miner penaltys are scaled up by
const PenaltyMultiplier = 3

type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.AwardBlockReward,
		3:                         a.ThisEpochReward,
		4:                         a.UpdateNetworkKPI,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.RewardActorCodeID
}

func (a Actor) IsSingleton() bool {
	return true
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

func (a Actor) Constructor(rt runtime.Runtime, currRealizedPower *abi.StoragePower) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)

	if currRealizedPower == nil {
		rt.Abortf(exitcode.ErrIllegalArgument, "argument should not be nil")
		return nil // linter does not understand abort exiting
	}
	st := ConstructState(*currRealizedPower)
	rt.StateCreate(st)
	return nil
}

//type AwardBlockRewardParams struct {
//	Miner     address.Address
//	Penalty   abi.TokenAmount // penalty for including bad messages in a block, >= 0
//	GasReward abi.TokenAmount // gas reward from all gas fees in a block, >= 0
//	WinCount  int64           // number of reward units won, > 0
//}
type AwardBlockRewardParams = reward0.AwardBlockRewardParams

// Awards a reward to a block producer.
// This method is called only by the system actor, implicitly, as the last message in the evaluation of a block.
// The system actor thus computes the parameters and attached value.
//
// The reward includes two components:
// - the epoch block reward, computed and paid from the reward actor's balance,
// - the block gas reward, expected to be transferred to the reward actor with this invocation.
//
// The reward is reduced before the residual is credited to the block producer, by:
// - a penalty amount, provided as a parameter, which is burnt,
func (a Actor) AwardBlockReward(rt runtime.Runtime, params *AwardBlockRewardParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)
	priorBalance := rt.CurrentBalance()
	if params.Penalty.LessThan(big.Zero()) {
		rt.Abortf(exitcode.ErrIllegalArgument, "negative penalty %v", params.Penalty)
	}
	if params.GasReward.LessThan(big.Zero()) {
		rt.Abortf(exitcode.ErrIllegalArgument, "negative gas reward %v", params.GasReward)
	}
	if priorBalance.LessThan(params.GasReward) {
		rt.Abortf(exitcode.ErrIllegalState, "actor current balance %v insufficient to pay gas reward %v",
			priorBalance, params.GasReward)
	}
	if params.WinCount <= 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid win count %d", params.WinCount)
	}

	minerAddr, ok := rt.ResolveAddress(params.Miner)
	if !ok {
		rt.Abortf(exitcode.ErrNotFound, "failed to resolve given owner address")
	}
	// The miner penalty is scaled up by a factor of PenaltyMultiplier
	penalty := big.Mul(big.NewInt(PenaltyMultiplier), params.Penalty)
	totalReward := big.Zero()
	var st State
	rt.StateTransaction(&st, func() {
		blockReward := big.Mul(st.ThisEpochReward, big.NewInt(params.WinCount))
		blockReward = big.Div(blockReward, big.NewInt(builtin.ExpectedLeadersPerEpoch))
		totalReward = big.Add(blockReward, params.GasReward)
		currBalance := rt.CurrentBalance()
		if totalReward.GreaterThan(currBalance) {
			rt.Log(rtt.WARN, "reward actor balance %d below totalReward expected %d, paying out rest of balance", currBalance, totalReward)
			totalReward = currBalance

			blockReward = big.Sub(totalReward, params.GasReward)
			// Since we have already asserted the balance is greater than gas reward blockReward is >= 0
			AssertMsg(blockReward.GreaterThanEqual(big.Zero()), "programming error, block reward is %v below zero", blockReward)
		}
		st.TotalStoragePowerReward = big.Add(st.TotalStoragePowerReward, blockReward)
	})

	AssertMsg(totalReward.LessThanEqual(priorBalance), "reward %v exceeds balance %v", totalReward, priorBalance)

	// if this fails, we can assume the miner is responsible and avoid failing here.
	rewardParams := builtin.ApplyRewardParams{
		Reward:  totalReward,
		Penalty: penalty,
	}
	code := rt.Send(minerAddr, builtin.MethodsMiner.ApplyRewards, &rewardParams, totalReward, &builtin.Discard{})
	if !code.IsSuccess() {
		rt.Log(rtt.ERROR, "failed to send ApplyRewards call to the miner actor with funds: %v, code: %v", totalReward, code)
		code := rt.Send(builtin.BurntFundsActorAddr, builtin.MethodSend, nil, totalReward, &builtin.Discard{})
		if !code.IsSuccess() {
			rt.Log(rtt.ERROR, "failed to send unsent reward to the burnt funds actor, code: %v", code)
		}
	}

	return nil
}

// Changed since v0:
// - removed ThisEpochReward (unsmoothed)
type ThisEpochRewardReturn struct {
	ThisEpochRewardSmoothed smoothing.FilterEstimate
	ThisEpochBaselinePower  abi.StoragePower
}

// The award value used for the current epoch, updated at the end of an epoch
// through cron tick.  In the case previous epochs were null blocks this
// is the reward value as calculated at the last non-null epoch.
func (a Actor) ThisEpochReward(rt runtime.Runtime, _ *abi.EmptyValue) *ThisEpochRewardReturn {
	rt.ValidateImmediateCallerAcceptAny()

	var st State
	rt.StateReadonly(&st)
	return &ThisEpochRewardReturn{
		ThisEpochRewardSmoothed: st.ThisEpochRewardSmoothed,
		ThisEpochBaselinePower:  st.ThisEpochBaselinePower,
	}
}

// Called at the end of each epoch by the power actor (in turn by its cron hook).
// This is only invoked for non-empty tipsets, but catches up any number of null
// epochs to compute the next epoch reward.
func (a Actor) UpdateNetworkKPI(rt runtime.Runtime, currRealizedPower *abi.StoragePower) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.StoragePowerActorAddr)
	if currRealizedPower == nil {
		rt.Abortf(exitcode.ErrIllegalArgument, "arugment should not be nil")
	}

	var st State
	rt.StateTransaction(&st, func() {
		prev := st.Epoch
		// if there were null runs catch up the computation until
		// st.Epoch == rt.CurrEpoch()
		for st.Epoch < rt.CurrEpoch() {
			// Update to next epoch to process null rounds
			st.updateToNextEpoch(*currRealizedPower)
		}

		st.updateToNextEpochWithReward(*currRealizedPower)
		// only update smoothed estimates after updating reward and epoch
		st.updateSmoothedEstimates(st.Epoch - prev)
	})
	return nil
}
```

#### 2.3.6.4[AccountActor](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.accountactor)

该`AccountActor`负责用户帐户。帐户参与者不是由创建的`InitActor`，但系统会调用其构造函数。通过向公共密钥样式的地址发送消息来创建帐户参与者。地址必须为`BLS`或`SECP`，否则应该存在退出错误。帐户参与者正在使用新的参与者地址更新状态树。

[例：](https://spec.filecoin.io/#example-)

```go
package account

import (
	addr "github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/exitcode"
	"github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v2/actors/builtin"
	"github.com/filecoin-project/specs-actors/v2/actors/runtime"
)

type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		1: a.Constructor,
		2: a.PubkeyAddress,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.AccountActorCodeID
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

type State struct {
	Address addr.Address
}

func (a Actor) Constructor(rt runtime.Runtime, address *addr.Address) *abi.EmptyValue {
	// Account actors are created implicitly by sending a message to a pubkey-style address.
	// This constructor is not invoked by the InitActor, but by the system.
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)
	switch address.Protocol() {
	case addr.SECP256K1:
	case addr.BLS:
		break // ok
	default:
		rt.Abortf(exitcode.ErrIllegalArgument, "address must use BLS or SECP protocol, got %v", address.Protocol())
	}
	st := State{Address: *address}
	rt.StateCreate(&st)
	return nil
}

// Fetches the pubkey-type address from this actor.
func (a Actor) PubkeyAddress(rt runtime.Runtime, _ *abi.EmptyValue) *addr.Address {
	rt.ValidateImmediateCallerAcceptAny()
	var st State
	rt.StateReadonly(&st)
	return &st.Address
}
```

### 2.3.7[VM解释器-消息调用（外部VM）](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter)

VM解释器根据提示集在其父状态上的提示集协调消息的执行，从而产生新状态和一系列消息回执。此新状态的CID和收据集合的CID包含在后续纪元的块中，这些纪元必须同意这些CID才能形成新的提示集。

每个状态更改都由消息的执行来驱动。提示集中所有块中的消息必须执行才能产生下一个状态。来自第一个块的所有消息均在技巧集中的第二个和后续块的消息之前执行。对于每个块，首先执行BLS聚合的消息，然后执行SECP签名的消息。

#### 2.3.7.1[隐式消息](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter.implicit-messages)

除了显式包含在每个块中的消息之外，隐含消息还会在每个时期对状态进行一些更改。隐式消息不在节点之间传输，而是由解释器在评估时构造的。

对于提示集中的每个块，隐式消息：

- 调用区块生产者的矿工演员来处理（已验证的）选举PoSt提交，作为区块中的第一条消息；
- 调用奖励参与者将区块奖励支付给矿工的所有者帐户，作为区块中的最终消息；

对于每个提示集，一个隐式消息：

- 调用cron actor来处理自动支票和付款，作为提示集中的最后一条消息。

所有隐式消息的构造`From`地址都是杰出的系统帐户参与者。他们将汽油价格指定为零，但必须包含在计算中。为了计算新状态，它们必须成功（退出代码为零）。隐式邮件的收据不包括在收据列表中；只有明确的消息才有明确的回执。

#### 2.3.7.2[煤气费](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter.gas-payments)

在大多数情况下，消息的发送者向产生包含该消息的块的矿工支付执行该消息所需的汽油费。

执行该消息后，每次执行该消息所产生的汽油费将立即支付给矿工所有者帐户。所获得的集体奖励或汽油费没有任何负担：两者都可以立即花费。

#### 2.3.7.3[邮件重复](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter.duplicate-messages)

由于不同的矿工在同一时期产生区块，因此单个提示集中的多个区块可能包含相同的消息（由相同的CID标识）。发生这种情况时，仅在第一次按提示集的规范顺序遇到该消息时才对其进行处理。消息的后续实例将被忽略，不会导致任何状态突变，产生收据或向区块生产者支付费用。

因此，总结了提示集的执行顺序：

- 为第一块支付奖励
- 处理第一块的选举职位
- 第一个块的消息（SECP之前的BLS）
- 支付第二块奖励
- 处理第二个区块的选举职位
- 第二个块的消息（SECP之前的BLS，跳过任何已经遇到的消息）
- `[... subsequent blocks ...]`
- 定时刻度

#### 2.3.7.4[消息有效性和失败](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter.message-validity-and-failure)

有效块中的每个消息都可以被处理并产生收据（请注意，块有效性表示所有消息在语法上均有效–请参阅 [消息语法](https://spec.filecoin.io/#section-systems.filecoin_vm.message.message-syntax-validation)–并正确签名）。但是，执行成功与否取决于消息所应用的状态。如果消息执行失败，则相应的收据将携带非零的退出代码。

如果消息由于可以合理地归因于矿工的原因而失败，包括在父状态中永远不可能成功的消息，或者由于发件人缺乏资金来支付最大消息成本，则矿工将通过烧钱来支付罚款煤气费（而不是发送方向大宗矿工支付的费用）。

消息失败导致的唯一状态更改是：

- 发送方的增量`CallSeqNum`，并从发送方向包含消息的区块矿主支付汽油费；要么
- 罚款等于失败消息的汽油费，由矿工烧掉（发件人未`CallSeqNum`更改）。

如果处于紧接的先前状态，则消息执行将失败：

- 该`From`演员不存在于该州（受到矿工处罚），
- 该`From`演员是不是帐号演员（的矿工处罚），
- 该`CallSeqNum`消息不匹配`CallSeqNum`的的`From`演员（的矿工处罚），
- 的`From`演员不具有足够的平衡，以覆盖消息的总和`Value`加上最大气体成本，`GasLimit * GasPrice`（矿工处罚），
- 该参与者不在`To`状态中，并且该`To`地址不是pubkey样式的地址，
- 该`To`actor存在（或作为帐户隐式创建），但是没有对应于非零的方法`MethodNum`，
- 反序列化`Params`不是长度匹配数组`To`actor的`MethodNum`方法的数组，
- 反序列化`Params`对于`To`actor的`MethodNum`方法指定的类型无效，
- 被调用的方法消耗的气体多于`GasLimit`允许的量，
- 调用的方法以非零代码（通过`Runtime.Abort()`）退出，或者
- 由于上述任何原因，接收方发送的任何内部消息都会失败。

请注意，如果`To`参与者不在状态中并且该地址是有效`H(pubkey)`地址，则它将被创建为帐户参与者。

## 2.4[区块链](https://spec.filecoin.io/#section-systems.filecoin_blockchain)

Filecoin区块链是一种分布式虚拟机，可以达成共识，处理消息，进行存储帐户并维护Filecoin协议中的安全性。它是链接Filecoin系统中各种参与者的主要界面。

Filecoin区块链系统包括：

- 一个 [消息池](https://spec.filecoin.io/#section-systems.filecoin_blockchain.message_pool)子系统节点使用跟踪和消息传播矿工已经宣布他们要在blockchain包括。
- 用于解释和执行消息以更新系统状态的 [虚拟](https://spec.filecoin.io/#section-systems.filecoin_vm)机子系统。
- 一 [国树](https://spec.filecoin.io/#section-systems.filecoin_vm.state_tree)，其管理的创建和状态的树木（系统状态）从给定的子链确定性VM产生的维护子系统。
- 甲 [链同步（ChainSync）](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync) susbystem验证消息块该轨道和传播，保持套候选链在其上可以矿工和矿上的传入块运行语法验证。
- 一个 [存储功率共识](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus)子系统，该子系统跟踪给定链的存储状态（即 [Storage Subystem](https://spec.filecoin.io/#section-systems.filecoin_mining.storage_mining)），并帮助区块链系统选择要扩展的子链并将其包括在子链中。

区块链系统还包括：

- 一个 [链经理](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.chain_manager)，它保持给定链的状态，提供设施等blockchain子系统将在顺序查询有关最新的链路状态来运行，并确保进入块纳入链之前在语义验证。
- 阿 [块生产者](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block_producer)被称为在一个成功的领导人选举的事件，以产生一个新的块，将其转发到所述同步装置为传播之前扩展当前最重链。

从高层次来看，Filecoin区块链通过连续几轮的领导人选举而增长，在选举中，许多矿工被选举产生一个区块，将其纳入链中将为他们赢得区块奖励。Filecoin的区块链依靠存储能力运行。也就是说，矿工通过其共识算法来确定要开采的子链取决于该子链的存储量。在高层，“ [存储功率共识”](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus)子系统维护一个*功率表*，该*表*跟踪[存储矿工参与者](https://spec.filecoin.io/#section-systems.filecoin_mining.storage_mining)通过*部门承诺*和*时空证明为*网络贡献的存储量 。

### 2.4.1[积木](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct)

区块是Filecoin区块链的主要单元，大多数其他区块链也是如此。阻止消息与提示集直接链接，提示集是阻止消息的组，本节稍后将对此进行详细介绍。在下文中，我们讨论Block消息的主要结构以及Filecoin区块链中验证Block消息的过程。

#### 2.4.1.1[块](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block)

区块是Filecoin区块链的主要单元。

Filecoin区块链中的Block结构包括：i）Block Header，ii）Block内的消息列表，以及iii）Signed消息。这在`FullBlock`抽象内部表示。该消息指示要应用的必需的一组更改，以达到链的确定性状态。

该块的Lotus实现具有以下内容`struct`：

[示例：FullBlock](https://spec.filecoin.io/#example-fullblock)

```go
type FullBlock struct {
	Header        *BlockHeader
	BlsMessages   []*Message
	SecpkMessages []*SignedMessage
}
```

> **注意**
> 块在功能上与Filecoin协议中的块头相同。虽然块标题包含指向完整系统状态，消息和消息回执的Merkle链接，但可以将块视为该信息的完整集合（不仅是Merkle根，还包括状态树的完整数据，消息树，收据树等）。由于完整块的大小很大，因此Filecoin区块链由块头而不是完整块组成。我们经常使用这些术语，`block`并且`block header`可以互换使用。

A`BlockHeader`是块的规范表示。BlockHeader在矿工节点之间传播。从blockcheader消息中，矿工拥有所有必需的信息，以应用关联`FullBlock`的状态并更新链。为了做到这一点，`BlockHeader`下面显示了需要包含的最少信息项集，其中包括：矿工的地址，票证， [时空证明](https://spec.filecoin.io/#section-algorithms.pos.post)，此块所在的父母的CID从IPLD DAG以及消息自身的CID演变而来。

块标头的Lotus实现具有以下`struct`：

[示例：BlockHeader](https://spec.filecoin.io/#example-blockheader)

```go
type BlockHeader struct {
	Miner address.Address // 0

	Ticket *Ticket // 1

	ElectionProof *ElectionProof // 2

	BeaconEntries []BeaconEntry // 3

	WinPoStProof []proof2.PoStProof // 4

	Parents []cid.Cid // 5

	ParentWeight BigInt // 6

	Height abi.ChainEpoch // 7

	ParentStateRoot cid.Cid // 8

	ParentMessageReceipts cid.Cid // 8

	Messages cid.Cid // 10

	BLSAggregate *crypto.Signature // 11

	Timestamp uint64 // 12

	BlockSig *crypto.Signature // 13

	ForkSignaling uint64 // 14

	// ParentBaseFee is the base fee after executing parent tipset
	ParentBaseFee abi.TokenAmount // 15

	// internal
	validated bool // true if the signature has been validated
}
```

[示例：票证](https://spec.filecoin.io/#example-ticket)

```go
type Ticket struct {
	VRFProof []byte
}
```

[示例：ElectionProof](https://spec.filecoin.io/#example-electionproof)

```go
type ElectionProof struct {
	WinCount int64
	VRFProof []byte
}
```

[示例：BeaconEntry](https://spec.filecoin.io/#example-beaconentry)

```go
type BeaconEntry struct {
	Round uint64
	Data  []byte
}
```

该`BlockHeader`结构必须引用当前回合的TicketWinner，以确保将正确的获胜者传递给 [ChainSync](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync)。

```go
func IsTicketWinner(vrfTicket []byte, mypow BigInt, totpow BigInt) bool
```

该`Message`结构必须包括源（`From`）和目标（`To`）地址，a`Nonce`和`GasPrice`。

消息的Lotus实现具有以下结构：

[示例：消息](https://spec.filecoin.io/#example-message)

```go
type Message struct {
	Version uint64

	To   address.Address
	From address.Address

	Nonce uint64

	Value abi.TokenAmount

	GasLimit   int64
	GasFeeCap  abi.TokenAmount
	GasPremium abi.TokenAmount

	Method abi.MethodNum
	Params []byte
}
```

在将消息传递到[链同步逻辑](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync)之前，还将对其进行验证 ：

[示例：ValidForBlockInclusion](https://spec.filecoin.io/#example-validforblockinclusion)

```go
func (m *Message) ValidForBlockInclusion(minGas int64) error {
	if m.Version != 0 {
		return xerrors.New("'Version' unsupported")
	}

	if m.To == address.Undef {
		return xerrors.New("'To' address cannot be empty")
	}

	if m.From == address.Undef {
		return xerrors.New("'From' address cannot be empty")
	}

	if m.Value.Int == nil {
		return xerrors.New("'Value' cannot be nil")
	}

	if m.Value.LessThan(big.Zero()) {
		return xerrors.New("'Value' field cannot be negative")
	}

	if m.Value.GreaterThan(TotalFilecoinInt) {
		return xerrors.New("'Value' field cannot be greater than total filecoin supply")
	}

	if m.GasFeeCap.Int == nil {
		return xerrors.New("'GasFeeCap' cannot be nil")
	}

	if m.GasFeeCap.LessThan(big.Zero()) {
		return xerrors.New("'GasFeeCap' field cannot be negative")
	}

	if m.GasPremium.Int == nil {
		return xerrors.New("'GasPremium' cannot be nil")
	}

	if m.GasPremium.LessThan(big.Zero()) {
		return xerrors.New("'GasPremium' field cannot be negative")
	}

	if m.GasPremium.GreaterThan(m.GasFeeCap) {
		return xerrors.New("'GasFeeCap' less than 'GasPremium'")
	}

	if m.GasLimit > build.BlockGasLimit {
		return xerrors.New("'GasLimit' field cannot be greater than a block's gas limit")
	}

	// since prices might vary with time, this is technically semantic validation
	if m.GasLimit < minGas {
		return xerrors.Errorf("'GasLimit' field cannot be less than the cost of storing a message on chain %d < %d", m.GasLimit, minGas)
	}

	return nil
}
```

##### 2.4.1.1.1[块语法验证](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block.block-syntax-validation)

语法验证是指应在*不*参考外部信息（例如父状态树）的*情况下*对块及其消息执行的验证。这种验证类型有时称为*静态验证*。

无效的块不得作为父对象传输或引用。

语法上有效的块头必须解码为与以下定义匹配的字段，必须是有效的CBOR PubSub`BlockMsg`消息，并且必须具有：

- 在1和`5*ec.ExpectedLeaders` `Parents`CID之间，如果`Epoch`大于零（否则为空`Parents`），

- 一个非负`ParentWeight`，

- 少于或等于`BlockMessageLimit`消息数，

- 封装在`MsgMeta`结构中的聚合消息CID，序列化为`Messages`块头中的CID，

- 一个`Miner`是ID地址的地址。`Address`块头中的Miner应该存在，并与当前链状态下的公共密钥地址相对应。

- `BlockSig`属于矿工检索到的公钥地址的块签名（）

- 一个非负`Epoch`，

- 一个积极的`Timestamp`，

- 一个`Ticket`非空`VRFResult`，

- ```
  ElectionPoStOutput
  ```

   包含：

  - 一个`Candidates`介于1和`EC.ExpectedLeaders`值之间（含）的数组，
  - 一个非空`PoStRandomness`字段，
  - 一个非空`Proof`字段，

- 一个非空`ForkSignal`字段。

句法有效的完整块必须具有：

- 所有引用的消息在语法上均有效，
- 所有引用的父母收据在语法上均有效，
- 块头和包含的消息的序列化大小之和不大于`block.BlockMaxSize`，
- 所有显式消息的气体限制总和不大于`block.BlockGasLimit`。

请注意，块签名的验证需要从父提示集状态访问矿工的地址和公钥，因此签名验证是语义验证的一部分。同样，消息签名验证要求查找与`From`处于块的父状态的每个消息的帐户执行者相关联的公钥。

##### 2.4.1.1.2[阻止语义验证](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block.block-semantic-validation)

语义验证是指需要引用块头和消息本身之外的信息的验证。语义验证与构建块的父提示集和状态有关。

为了进行语义验证，`FullBlock`必须从接收到的块头中提取其Filecoin消息进行组装。可以从网络中检索阻止消息CID，并将其解码为有效的CBOR `Message`/ `SignedMessage`。

在Lotus实现中，模块的语义验证由`Syncer`模块执行：

[示例：ValidateBlock](https://spec.filecoin.io/#example-validateblock)

ValidateBlock应该与spec.validation.md中的“语义验证”匹配

```go
func (syncer *Syncer) ValidateBlock(ctx context.Context, b *types.FullBlock, useCache bool) (err error) {
	defer func() {
		// b.Cid() could panic for empty blocks that are used in tests.
		if rerr := recover(); rerr != nil {
			err = xerrors.Errorf("validate block panic: %w", rerr)
			return
		}
	}()

	if useCache {
		isValidated, err := syncer.store.IsBlockValidated(ctx, b.Cid())
		if err != nil {
			return xerrors.Errorf("check block validation cache %s: %w", b.Cid(), err)
		}

		if isValidated {
			return nil
		}
	}

	validationStart := build.Clock.Now()
	defer func() {
		stats.Record(ctx, metrics.BlockValidationDurationMilliseconds.M(metrics.SinceInMilliseconds(validationStart)))
		log.Infow("block validation", "took", time.Since(validationStart), "height", b.Header.Height, "age", time.Since(time.Unix(int64(b.Header.Timestamp), 0)))
	}()

	ctx, span := trace.StartSpan(ctx, "validateBlock")
	defer span.End()

	if err := blockSanityChecks(b.Header); err != nil {
		return xerrors.Errorf("incoming header failed basic sanity checks: %w", err)
	}

	h := b.Header

	baseTs, err := syncer.store.LoadTipSet(types.NewTipSetKey(h.Parents...))
	if err != nil {
		return xerrors.Errorf("load parent tipset failed (%s): %w", h.Parents, err)
	}

	lbts, lbst, err := stmgr.GetLookbackTipSetForRound(ctx, syncer.sm, baseTs, h.Height)
	if err != nil {
		return xerrors.Errorf("failed to get lookback tipset for block: %w", err)
	}

	prevBeacon, err := syncer.store.GetLatestBeaconEntry(baseTs)
	if err != nil {
		return xerrors.Errorf("failed to get latest beacon entry: %w", err)
	}

	// fast checks first
	nulls := h.Height - (baseTs.Height() + 1)
	if tgtTs := baseTs.MinTimestamp() + build.BlockDelaySecs*uint64(nulls+1); h.Timestamp != tgtTs {
		return xerrors.Errorf("block has wrong timestamp: %d != %d", h.Timestamp, tgtTs)
	}

	now := uint64(build.Clock.Now().Unix())
	if h.Timestamp > now+build.AllowableClockDriftSecs {
		return xerrors.Errorf("block was from the future (now=%d, blk=%d): %w", now, h.Timestamp, ErrTemporal)
	}
	if h.Timestamp > now {
		log.Warn("Got block from the future, but within threshold", h.Timestamp, build.Clock.Now().Unix())
	}

	msgsCheck := async.Err(func() error {
		if err := syncer.checkBlockMessages(ctx, b, baseTs); err != nil {
			return xerrors.Errorf("block had invalid messages: %w", err)
		}
		return nil
	})

	minerCheck := async.Err(func() error {
		if err := syncer.minerIsValid(ctx, h.Miner, baseTs); err != nil {
			return xerrors.Errorf("minerIsValid failed: %w", err)
		}
		return nil
	})

	baseFeeCheck := async.Err(func() error {
		baseFee, err := syncer.store.ComputeBaseFee(ctx, baseTs)
		if err != nil {
			return xerrors.Errorf("computing base fee: %w", err)
		}
		if types.BigCmp(baseFee, b.Header.ParentBaseFee) != 0 {
			return xerrors.Errorf("base fee doesn't match: %s (header) != %s (computed)",
				b.Header.ParentBaseFee, baseFee)
		}
		return nil
	})
	pweight, err := syncer.store.Weight(ctx, baseTs)
	if err != nil {
		return xerrors.Errorf("getting parent weight: %w", err)
	}

	if types.BigCmp(pweight, b.Header.ParentWeight) != 0 {
		return xerrors.Errorf("parrent weight different: %s (header) != %s (computed)",
			b.Header.ParentWeight, pweight)
	}

	stateRootCheck := async.Err(func() error {
		stateroot, precp, err := syncer.sm.TipSetState(ctx, baseTs)
		if err != nil {
			return xerrors.Errorf("get tipsetstate(%d, %s) failed: %w", h.Height, h.Parents, err)
		}

		if stateroot != h.ParentStateRoot {
			msgs, err := syncer.store.MessagesForTipset(baseTs)
			if err != nil {
				log.Error("failed to load messages for tipset during tipset state mismatch error: ", err)
			} else {
				log.Warn("Messages for tipset with mismatching state:")
				for i, m := range msgs {
					mm := m.VMMessage()
					log.Warnf("Message[%d]: from=%s to=%s method=%d params=%x", i, mm.From, mm.To, mm.Method, mm.Params)
				}
			}

			return xerrors.Errorf("parent state root did not match computed state (%s != %s)", stateroot, h.ParentStateRoot)
		}

		if precp != h.ParentMessageReceipts {
			return xerrors.Errorf("parent receipts root did not match computed value (%s != %s)", precp, h.ParentMessageReceipts)
		}

		return nil
	})

	// Stuff that needs worker address
	waddr, err := stmgr.GetMinerWorkerRaw(ctx, syncer.sm, lbst, h.Miner)
	if err != nil {
		return xerrors.Errorf("GetMinerWorkerRaw failed: %w", err)
	}

	winnerCheck := async.Err(func() error {
		if h.ElectionProof.WinCount < 1 {
			return xerrors.Errorf("block is not claiming to be a winner")
		}

		eligible, err := stmgr.MinerEligibleToMine(ctx, syncer.sm, h.Miner, baseTs, lbts)
		if err != nil {
			return xerrors.Errorf("determining if miner has min power failed: %w", err)
		}

		if !eligible {
			return xerrors.New("block's miner is ineligible to mine")
		}

		rBeacon := *prevBeacon
		if len(h.BeaconEntries) != 0 {
			rBeacon = h.BeaconEntries[len(h.BeaconEntries)-1]
		}
		buf := new(bytes.Buffer)
		if err := h.Miner.MarshalCBOR(buf); err != nil {
			return xerrors.Errorf("failed to marshal miner address to cbor: %w", err)
		}

		vrfBase, err := store.DrawRandomness(rBeacon.Data, crypto.DomainSeparationTag_ElectionProofProduction, h.Height, buf.Bytes())
		if err != nil {
			return xerrors.Errorf("could not draw randomness: %w", err)
		}

		if err := VerifyElectionPoStVRF(ctx, waddr, vrfBase, h.ElectionProof.VRFProof); err != nil {
			return xerrors.Errorf("validating block election proof failed: %w", err)
		}

		slashed, err := stmgr.GetMinerSlashed(ctx, syncer.sm, baseTs, h.Miner)
		if err != nil {
			return xerrors.Errorf("failed to check if block miner was slashed: %w", err)
		}

		if slashed {
			return xerrors.Errorf("received block was from slashed or invalid miner")
		}

		mpow, tpow, _, err := stmgr.GetPowerRaw(ctx, syncer.sm, lbst, h.Miner)
		if err != nil {
			return xerrors.Errorf("failed getting power: %w", err)
		}

		j := h.ElectionProof.ComputeWinCount(mpow.QualityAdjPower, tpow.QualityAdjPower)
		if h.ElectionProof.WinCount != j {
			return xerrors.Errorf("miner claims wrong number of wins: miner: %d, computed: %d", h.ElectionProof.WinCount, j)
		}

		return nil
	})

	blockSigCheck := async.Err(func() error {
		if err := sigs.CheckBlockSignature(ctx, h, waddr); err != nil {
			return xerrors.Errorf("check block signature failed: %w", err)
		}
		return nil
	})

	beaconValuesCheck := async.Err(func() error {
		if os.Getenv("LOTUS_IGNORE_DRAND") == "_yes_" {
			return nil
		}

		if err := beacon.ValidateBlockValues(syncer.beacon, h, baseTs.Height(), *prevBeacon); err != nil {
			return xerrors.Errorf("failed to validate blocks random beacon values: %w", err)
		}
		return nil
	})

	tktsCheck := async.Err(func() error {
		buf := new(bytes.Buffer)
		if err := h.Miner.MarshalCBOR(buf); err != nil {
			return xerrors.Errorf("failed to marshal miner address to cbor: %w", err)
		}

		if h.Height > build.UpgradeSmokeHeight {
			buf.Write(baseTs.MinTicket().VRFProof)
		}

		beaconBase := *prevBeacon
		if len(h.BeaconEntries) != 0 {
			beaconBase = h.BeaconEntries[len(h.BeaconEntries)-1]
		}

		vrfBase, err := store.DrawRandomness(beaconBase.Data, crypto.DomainSeparationTag_TicketProduction, h.Height-build.TicketRandomnessLookback, buf.Bytes())
		if err != nil {
			return xerrors.Errorf("failed to compute vrf base for ticket: %w", err)
		}

		err = VerifyElectionPoStVRF(ctx, waddr, vrfBase, h.Ticket.VRFProof)
		if err != nil {
			return xerrors.Errorf("validating block tickets failed: %w", err)
		}
		return nil
	})

	wproofCheck := async.Err(func() error {
		if err := syncer.VerifyWinningPoStProof(ctx, h, *prevBeacon, lbst, waddr); err != nil {
			return xerrors.Errorf("invalid election post: %w", err)
		}
		return nil
	})

	await := []async.ErrorFuture{
		minerCheck,
		tktsCheck,
		blockSigCheck,
		beaconValuesCheck,
		wproofCheck,
		winnerCheck,
		msgsCheck,
		baseFeeCheck,
		stateRootCheck,
	}

	var merr error
	for _, fut := range await {
		if err := fut.AwaitContext(ctx); err != nil {
			merr = multierror.Append(merr, err)
		}
	}
	if merr != nil {
		mulErr := merr.(*multierror.Error)
		mulErr.ErrorFormat = func(es []error) string {
			if len(es) == 1 {
				return fmt.Sprintf("1 error occurred:\n\t* %+v\n\n", es[0])
			}

			points := make([]string, len(es))
			for i, err := range es {
				points[i] = fmt.Sprintf("* %+v", err)
			}

			return fmt.Sprintf(
				"%d errors occurred:\n\t%s\n\n",
				len(es), strings.Join(points, "\n\t"))
		}
		return mulErr
	}

	if useCache {
		if err := syncer.store.MarkBlockAsValidated(ctx, b.Cid()); err != nil {
			return xerrors.Errorf("caching block validation %s: %w", b.Cid(), err)
		}
	}

	return nil
}
```

邮件是通过检索的`Syncer`。遵循以下两个步骤`Syncer`：1-`FullTipSet`用先前收到的单个块组装一个填充块。该数据块`ParentWeight`大于最重的提示集（第一个数据块）中的数据块。2-从收到的块中检索所有提示集，直到我们的链。验证扩展到这些提示集中的每个块。验证应确保：-信标整体按其轮次排序。-提示集`Parents`CID与通过BlockSync获取的父提示集匹配。

语义上有效的块必须满足以下所有要求。

**`Parents`-有关**

- `Parents`按其标题的字典顺序列出`Ticket`。
- `ParentStateRoot`块的CID与从父[Tipset](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.tipset)计算的状态CID匹配 。
- `ParentState` 将通过执行父提示集的消息（由VM解释器定义）产生的状态树与该提示集的父状态相匹配。
- `ParentMessageReceipts`标识父提示集执行产生的收据列表，并为父提示集的每条唯一消息提供一张收据。换句话说，块的`ParentMessageReceipts`CID与从父提示集计算的收据CID匹配。
- `ParentWeight` 匹配链的权重，直到并包括父提示集。

**与时间有关**

- ```
  Epoch
  ```

  大于其

  ```
  Parents
  ```

  ，并且

  - 将来不会根据节点当前时间的本地时钟读数，

    - 在适当的时期之前，不应拒绝具有未来时期的模块，但不应对其进行评估（验证或包含在提示集中）

  - 不过去比软终结更远通过SPC定义的 

    终局性

    ，

    - 该规则仅在接收到新的八卦块（即从当前链头）接收时才适用，而不是在首次同步到链时。

- 在

  ```
  Timestamp
  ```

  包括以秒为单位的是：

  - 不得大于当前时间加上 `ΑllowableClockDriftSecs`
  - 不得小于前一个块的`Timestamp`加号`BlockDelay`（包括空块）
  - 具有创世块的时间戳，网络的锁定时间和锁定的所隐含的精确值`Epoch`。

**`Miner`-有关**

- 的`Miner`是在父tipset状态存储功率表活性。矿工的地址已注册在`Claims`Power Actor的HAMT中

- 对于

  ```
  TipSetState
  ```

  要验证的每个提示集，都应包含。

  - 提示集中的每个区块都应属于不同的矿工。

- 与消息的`From`地址关联的Actor存在，是帐户actor，其Nonce与消息Nonce匹配。

- 包括有效的证据，证明该矿工证明可以访问其面临挑战的部门的密封版本。为了实现这一目标：

  - 使用`WinningPoSt`域分隔标签为当前时期绘制随机性。
  - 根据绘制的随机性，获取此矿工在此时期面临挑战的扇区列表。

- 矿工不被削减`StoragePowerActor`。

**`Beacon`- ＆`Ticket`-相关**

- 有效期

  ```
  BeaconEntries
  ```

  应包括：

  - 检查中的每一个`BeaconEntries`都是消息的签名：`previousSignature || round`使用DRAND的公钥签名。
  - 包括从`MaxBeaconRoundForEpoch`低到高`prevEntry`（从上一个技巧集开始）之间的所有条目。

- 一个

  ```
  Ticket
  ```

  来自父tipset的块头最小票获得，

  - `Ticket.VRFResult`由`Miner`演员的工人帐户公钥有效签署，

- `ElectionProof Ticket`通过使用矿工的密钥检查BLS签名来正确计算得出。该`ElectionProof`票应该是中奖票。

**消息和签名相关**

- `secp256k1`邮件已通过其发送方（`From`）工作者帐户密钥正确签名，

- `BLSAggregate`包括一个签名，该签名使用该块的发送参与者的密钥来签名该块引用的所有BLS消息的CID数组。

- `Signature`包含来自块的`Miner`参与者的工作人员帐户公共密钥的块头的有效字段。

- 对于

  ```
  ValidForBlockInclusion()
  ```

  以下保持的每条消息：

  - 消息字段`Version`，`To`，`From`，`Value`，`GasPrice`，和`GasLimit`正确定义。
  - 消息`GasLimit`低于消息的最低气体成本（来自链高度和消息长度）。

- 对于其中的每个消息

  ```
  ApplyMessage
  ```

  （即在执行消息之前），以下保持：

  - 基本气体和价值检查在

    ```
    checkMessage()
    ```

    ：

    - 消息`GasLimit`大于零。
    - 消息`GasPrice`和`Value`已设置。

  - 消息的存储气体成本低于消息的`GasLimit`。

  - 消息`Nonce`与从消息`From`地址中检索到的Actor中的随机数匹配。

  - 消息的最大用气成本（从其`GasLimit`，`GasPrice`和派生`Value`）低于从消息`From`地址检索的Actor的余额。

  - 消息的传输`Value`处于从消息`From`地址中检索到的Actor的余额之下。

除了验证签名之外，没有对块中包含的消息进行语义验证的方法。如果块中包含的所有消息在语法上均有效，则可以执行它们并产生收据。

链同步系统可以分阶段执行语法和语义验证，以最大程度地减少不必要的资源消耗。

如果以上所有测试均成功，则将该块标记为已验证。最终，无效块不得进一步传播或验证为父节点。

#### 2.4.1.2[小费](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.tipset)

预期共识在每个时期概率选举出多个领导者，这意味着Filecoin链在每个时期可能包含零个或多个区块（每个选举的矿工一个）。来自同一时期的块被组装到提示集中。所述 [VM解释器](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter)通过在tipset执行的所有消息（包括在多于一个的块相同消息的重复数据删除之后）修改Filecoin状态树。

每个块都引用一个父提示集，并验证*该提示集的state*，同时提出要包含在当前时代的消息。除非将该块合并到提示集中，否则无法知道新块的消息所应用的状态。因此，孤立地从单个块中执行消息是没有意义的：只有执行了该块的技巧集中的所有消息后，才知道新的状态树。

一个有效的提示集包含一个非空的块集合，这些块具有不同的矿工，并且全部指定相同：

- `Epoch`
- `Parents`
- `ParentWeight`
- `StateRoot`
- `ReceiptsRoot`

提示集中的各个块按每个块票证中字节的字典顺序进行规范排序，从而与该块本身的CID字节断开联系。

由于网络传播延迟，在时间段N + 1中的矿工可能会从其父提示集中忽略在时间段N处挖掘的有效块。这不会使新生成的块无效，但是会降低其权重和成为EC[链选择](https://spec.filecoin.io/#section-algorithms.expected_consensus.chain-selection)功能定义的协议中规范链一部分的机会 。

块生产者应该协调他们如何选择要包含在块中的消息，以避免重复，从而从消息费用中最大化他们的预期收益（请参阅 [消息池](https://spec.filecoin.io/#section-systems.filecoin_blockchain.message_pool)）。

Lotus实现中的主要Tipset结构包括以下内容：

[示例：TipSet](https://spec.filecoin.io/#example-tipset)

```go
type TipSet struct {
	cids   []cid.Cid
	blks   []*BlockHeader
	height abi.ChainEpoch
}
```

Tipset的语义验证包括以下检查。

[示例：NewTipSet](https://spec.filecoin.io/#example-newtipset)

检查：

- 提示集由至少一个块组成。（由于每个提示集的可变块数由随机性决定，因此我们不施加上限。）
- 所有块都具有相同的高度。
- 所有块都具有相同的父代（它们具有相同的数量和匹配的CID）。

```go
func NewTipSet(blks []*BlockHeader) (*TipSet, error) {
	if len(blks) == 0 {
		return nil, xerrors.Errorf("NewTipSet called with zero length array of blocks")
	}

	sort.Slice(blks, tipsetSortFunc(blks))

	var ts TipSet
	ts.cids = []cid.Cid{blks[0].Cid()}
	ts.blks = blks
	for _, b := range blks[1:] {
		if b.Height != blks[0].Height {
			return nil, fmt.Errorf("cannot create tipset with mismatching heights")
		}

		if len(blks[0].Parents) != len(b.Parents) {
			return nil, fmt.Errorf("cannot create tipset with mismatching number of parents")
		}

		for i, cid := range b.Parents {
			if cid != blks[0].Parents[i] {
				return nil, fmt.Errorf("cannot create tipset with mismatching parents")
			}
		}

		ts.cids = append(ts.cids, b.Cid())

	}
	ts.height = blks[0].Height

	return &ts, nil
}
```

#### 2.4.1.3[连锁店经理](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.chain_manager)

所述*链经理*是在blockchain系统中的中心组件。它跟踪并更新给定节点接收到的竞争子链，以选择适当的区块链头：它在系统中知道的最重子链的最新块。

这样做，*链管理器*是中央子系统，它处理Filecoin节点中许多其他系统的簿记工作，并公开了供那些系统使用的便捷方法，从而使系统能够从链中抽样随机性，或查看哪个块已被占用。最近完成。

##### 2.4.1.3.1[延伸链](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.chain_manager.chain-extension)

###### 2.4.1.3.1.1[接收块接收](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.chain_manager.incoming-block-reception)

对于每个传入的块，即使未将传入的块添加到当前最重的提示集中，链管理器也应将其添加到它正在跟踪的适当子链中，或独立地对其进行跟踪，直到：

- 它能够通过接收该子链中的另一个块来添加到当前最重的子链中，或者
- 它可以丢弃它，因为该块是在最终确定之前开采的。

重要的是要注意，在最终确定之前，给定的子链可能会被替换为在给定回合中开采的另一个较重的子链。为了快速适应这种情况，链管理者必须维护和更新所有考虑到最终的子链。

链选择是Filecoin区块链工作方式的关键组成部分。简而言之，每个链都有相关的权重，这些权重说明了在其上开采的块数以及它们跟踪的功率（存储）。“选择[链”](https://spec.filecoin.io/#section-algorithms.expected_consensus.chain-selection)部分提供了有关选择工作原理的全部详细信息 。

**注释/建议：**

1. 为了简化某些验证检查，应该按高度和父集对块进行索引。这样，可以快速查询具有给定高度和普通父母的几组积木。
2. 在这些集中计算和缓存块的结果聚合状态可能也很有用，当检查有多个父级的块从哪个状态根开始时，这样可以节省额外的状态计算。
3. 建议将块保留在本地数据存储中，无论此时是否将其理解为最佳技巧-这是为了避免将来不得不重新提取相同的块。

###### 2.4.1.3.1.2[ChainTipsManager](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.chain_manager.chaintipsmanager)

链技巧管理器是Filecoin共识的子组件，负责跟踪Filecoin区块链的所有实时技巧，并跟踪当前的“最佳”技巧集。

```go
// Returns the ticket that is at round 'r' in the chain behind 'head'
func TicketFromRound(head Tipset, r Round) {}

// Returns the tipset that contains round r (Note: multiple rounds' worth of tickets may exist within a single block due to losing tickets being added to the eventually successfully generated block)
func TipsetFromRound(head Tipset, r Round) {}

// GetBestTipset returns the best known tipset. If the 'best' tipset hasn't changed, then this
// will return the previous best tipset.
func GetBestTipset()

// Adds the losing ticket to the chaintips manager so that blocks can be mined on top of it
func AddLosingTicket(parent Tipset, t Ticket)
```

#### 2.4.1.4[制片人](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block_producer)

##### 2.4.1.4.1[采矿块](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block_producer.mining-blocks)

如果已证明拥有满足[最小矿工规模](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.minimum-miner-size)阈值要求的储量，则向储能演员注册的矿工可以开始生成和检查选举票 。

为了做到这一点，矿工必须进行连锁验证，并跟踪收到的最新区块。矿工的新区块将基于前一个时期的父母。

###### 2.4.1.4.1.1[块创建](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block_producer.block-creation)

为一个时期生成一个块`H`需要等待该时期的信标输入并使用它来运行`GenerateElectionProof`。如果`WinCount`≥1（即，当矿工当选），相同的信标条目用于运行`WinningPoSt`。有了`ElectionProof`票证（的输出`GenerateElectionProof`）和`WinningPoSt`证明，矿工可以生产一个新的区块。

见 [VM解释](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter)为母体tipset评价的细节，并 [阻止](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block)对有效块标头值的约束。

要创建一个区块，合格的矿工必须计算一些字段：

- `Parents` -父提示集块的CID。

- `ParentWeight`-父链的权重（请参阅“ [链选择”](https://spec.filecoin.io/#section-algorithms.expected_consensus.chain-selection)）。

- `ParentState`-来自父提示集状态评估的状态根的CID（请参阅 [VM Interpreter](https://spec.filecoin.io/#section-systems.filecoin_vm.interpreter)）。

- `ParentMessageReceipts`-AMT根目录的CID，其中包含计算时产生的收据`ParentState`。

- `Epoch`-块的纪元，从该`Parents`纪元和生成该块所花费的纪元数得出。

- `Timestamp` -创建块时生成的Unix时间戳（以秒为单位）。

- `BeaconEntries`-从最后一个块开始生成的一组drand条目（请参阅 [Beacon Entries](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.beacon-entries)）。

- `Ticket`-从上一个纪元生成的新票证（请参见 [票证生成](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.randomness-ticket-generation)）。

- `Miner` -区块生产者的矿工演员地址。

- ```
  Messages
  ```

  \-

  ```
  TxMeta
  ```

  包含建议包含在新块中的消息的对象的CID ：

  - 从内存池中选择一组消息以包括在块中，以满足块大小和气体限制
  - 将邮件分为BLS签名邮件和secpk签名邮件
  - `TxMeta.BLSMessages`：包含裸笔`UnsignedMessage`的AMT根目录的CID
  - `TxMeta.SECPMessages`：一个AMT的根的CID包括`SignedMessage`小号

- `BeaconEntries`：从中导出随机性的信标条目列表

- `BLSAggregate` -使用BLS签名的块中所有消息的聚集签名。

- `Signature` -在区块标题的序列化表示形式（带有空签名）上具有矿工的工人帐户私钥的签名（还必须与票证签名匹配）。

- `ForkSignaling`-uint64标志用作信令分叉的一部分。默认情况下应设置为0。

注意，要生成有效块，无需评估要包含在块中的消息。矿工可能仍希望通过投机方式评估消息，以便进行优化，以包括将成功执行并支付最多汽油的消息。

产生积木时不评估积木奖励。在以下纪元的提示集中包含该区块时，将对其进行支付。

区块的签名确保了区块在传播后的完整性，因为与许多PoW区块链不同，发现中奖票证与区块生成无关。

###### 2.4.1.4.1.2[块广播](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block_producer.block-broadcast)

合格的矿工使用[GossipSub](https://spec.filecoin.io/#section-algorithms.gossip_sub) `/fil/blocks`主题将完成的区块传播到网络 ，并且假设一切都正确完成，则网络将接受它，其他矿工将在其之上进行挖掘，从而获得矿工的奖励。

矿工应在产生有效块后立即输出其有效块，否则冒着其他矿工冒着在EPOCH_CUTOFF之后接收该块且不将其包括在当前时期中的风险。

##### 2.4.1.4.2[块奖励](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block_producer.block-rewards)

集体奖励由[奖励演员](https://spec.filecoin.io/#section-systems.filecoin_vm.sysactors.rewardactor)处理 。在[Filecoin令牌](https://spec.filecoin.io/#section-systems.filecoin_token)部分讨论了 有关区块奖励的更多详细信息，在[矿工抵押品](https://spec.filecoin.io/#section-systems.filecoin_mining.miner_collaterals)部分讨论了有关区块奖励抵押的详细信息 。

### 2.4.[信息池](https://spec.filecoin.io/#section-systems.filecoin_blockchain.message_pool)

消息池，或者`mpool`或者`mempool`是Filecoin协议中的消息池。它充当Filecoin节点与用于链下消息传播的其他节点的对等网络之间的接口。节点使用消息池来维护它们要传输到Filecoin VM并添加到链中的一组消息（即，添加用于“链上”执行）。

为了使消息最终出现在区块链中，它首先必须位于消息池中。实际上，至少在Filecoin的Lotus实现中，没有中央消息池存储在某处。而是，消息池是一种抽象，并实现为网络中每个节点保留的消息列表。因此，当节点将新消息放入消息池时，该消息将使用libp2p的pubsub协议GossipSub传播到网络的其余部分。节点需要订阅相应的pubsub主题才能接收消息。

使用GossipSub进行消息传播不会立即发生，因此，在不同节点上的消息池可以同步之前存在一些滞后。实际上，在给消息池添加连续的消息流以及传播消息的延迟的情况下，消息池永远不会在网络中的所有节点之间同步。这不是系统的缺点，如不是短信池也*需要*通过网络进行同步。

消息池应具有定义的最大大小，以避免DoS攻击，在DoS攻击中，节点被垃圾邮件吞噬并耗尽内存。消息池的建议大小为5000条消息。

#### 2.4.2.1[讯息传播](https://spec.filecoin.io/#section-systems.filecoin_blockchain.message_pool.message_syncer)

消息池必须与libp2p pubsub [GossipSub](https://github.com/libp2p/specs/tree/master/pubsub/gossipsub)协议接口 。这是因为消息通过[GossipSub](https://github.com/libp2p/specs/tree/master/pubsub/gossipsub)传播了 相应的`/fil/msgs/` *主题*。参与网络的任何节点都会在相应的主题中宣布每个 [消息](https://spec.filecoin.io/#section-systems.filecoin_vm.message)`/fil/msgs/`。

有两个与消息和块相关的主要pubsub主题：i）`/fil/msgs/`携带消息的主题，以及ii）`/fil/blocks/`携带块的主题。该`/fil/msgs/`主题链接到`mpool`。流程如下：

1. 当客户想要在Filecoin网络中发送消息时，他们会将消息发布到`/fil/msgs/`主题。
2. 该消息使用GossipSub传播到网络中的所有其他节点，并最终出现在`mpool`所有矿工中。
3. 根据加密货币经济规则，某些矿工最终会从中`mpool`（与其他消息一起）挑选消息并将其包含在一个块中。
4. 矿工在`/fil/blocks/`pubsub主题中发布新挖掘的块，并且该块传播到网络中的所有节点（包括发布此块中包含的消息的节点）。

节点必须检查传入消息是否有效，即它们是否具有有效签名。如果该消息无效，则应将其丢弃并且不得转发。

GossipSub协议的更新的强化版本包括多种缓解攻击的策略。例如，当节点收到无效消息时，它将负*分数*分配给发送方对等方。对等分数不与其他节点共享，而是由其与之交互的所有其他对等点在每个对等点本地保存。如果对等方的得分下降到阈值以下，则将其从评分对等方的网格中排除。我们将在GossipSub部分中讨论有关这些设置的更多详细信息。完整的细节可以在[GossipSub规范中](https://github.com/libp2p/specs/tree/master/pubsub/gossipsub)找到 。

笔记：

- *资金检查：*重要的是要注意，`mpool`逻辑不是检查邮件发行者帐户中是否有足够的资金。矿工在将消息包含在块中之前会对此进行检查。
- *消息排序：*消息按照`mpool`到达矿工时遵循的加密经济规则，按矿工的顺序进行排序，以便矿工组成下一个区块。

#### 2.4.2.2[讯息储存](https://spec.filecoin.io/#section-systems.filecoin_blockchain.message_pool.message_storage)

如前所述，没有包含消息的中央池。相反，每个节点必须已为传入消息分配了内存。

### 2.4.3[链同步](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync)

区块链同步（“ sync”）是区块链系统的关键部分。它处理块和消息的检索和传播，因此负责分布式状态复制。因此，此过程对安全性至关重要-状态复制问题可能对区块链的运行产生严重影响。

当节点首次加入网络时，它会发现对等节点（通过上面讨论的对等节点发现），并加入`/fil/blocks`和`/fil/msgs`GossipSub主题。它侦听其他节点正在传播的新块。它选择一个块作为，`BestTargetHead`并开始同步从到此高度的区块链`TrustedCheckpoint`，默认情况下为`GenesisBlock`or `GenesisCheckpoint`。为了挑选`BestTargetHead`同伴，他们比较身高和体重的组合-这些值越高，区块出现在主链上的机会就越高。如果两个块的高度相同，则对等方应选择权重较高的一个。一旦对等方选择，`BestTargetHead`它将使用BlockSync协议来获取块并达到当前高度。从那一点开始`CHAIN_FOLLOW` 模式，它使用GossipSub接收新的块，或者，如果听到有关它尚未通过GossipSub接收的块的消息，则使用Bitswap。

#### 2.4.3.[ChainSync概述](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync.chainsync-overview)

`ChainSync`是Filecoin用来同步其区块链的协议。它特定于Filecoin在状态表示和共识规则中的选择，但是足够通用，可以服务于其他区块链。`ChainSync`是一组较小的协议，它们处理同步过程的不同部分。

在以下情况下，通常需要链同步：

1. 当节点首次加入网络并且需要在验证或扩展链之前达到当前状态时。
2. 当节点由于短暂断开而失去同步时。
3. 在正常操作期间，以跟上最新消息和块。

在这三种情况下，使用三种主要协议来实现同步。

- `GossipSub`是用于传播消息和块的libp2p pubsub协议。当节点需要与正在产生和传播的新块保持同步时，它主要用于以上第三步。
- `BlockSync` 用于同步链的特定部分，即从特定高度到特定高度。
- `hello`协议，当两个对等方首次“见面”时（即，他们第一次相互连接）使用。根据协议，他们交换链头。

另外，`Bitswap`当节点同步（“追赶”）但GossipSub无法将某些块传递到节点时，用于请求和接收块。最后，`GraphSync`可以用来获取区块链的一部分作为的更有效版本`Bitswap`。

Filecoin节点是libp2p节点，因此可以运行多种其他协议。与Filecoin中的其他任何内容一样，节点可以选择使用其他协议来获得结果。也就是说，节点必须实现`ChainSync`本规范中所述的版本，才能被视为Filecoin的实现。

#### 2.4.3.2[术语和概念](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync.terms-and-concepts)

- `LastCheckpoint``ChainSync`意识到的最后一个严格的面向社会共识的检查点。这个共识检查点定义了最小的终结性和最小的历史基础。 `ChainSync`接受`LastCheckpoint`并建立信念，永不背离其历史。
- `TargetHeads``BlockCIDs`代表块生产边缘的块的列表。这些是最新和最好的`ChainSync`知识。它们是“目标”头，因为 `ChainSync`将尝试与其同步。该列表按“成为最佳连锁店的可能性”排序。在这一点上，只需通过即可实现`ChainWeight`。
- `BestTargetHead``BlockCID`尝试与之同步的最佳链头。这是`TargetHeads`

#### 2.4.3.3[ChainSync状态机](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync.chainsync-state-machine)

在较高级别上，请`ChainSync`执行以下操作：

- 第1部分：验证内部状态（`INIT`以下状态）
  - 应该验证数据结构并验证本地链
  - 资源昂贵的验证可能会被跳过，节点自行承担风险
- 第2部分：引导至网络（`BOOTSTRAP`）
  - 步骤1.引导到网络，并获得一组“足够安全”的对等体（下面有更多详细信息）
  - 步骤2.引导至`GossipSub`渠道
- 第3部分：同步受信任的检查点状态（`SYNC_CHECKPOINT`）
  - 步骤1.以`TrustedCheckpoint`（默认为`GenesisCheckpoint`）开头。本`TrustedCheckpoint`不应该在软件进行验证，它应该由运营商进行验证。
  - 第2步。获取它所指向的块，以及该块的父母
  - 第3步。 `StateTree`
- 第4部分：追上链（`CHAIN_CATCHUP`）
  - 步骤1.保持一组`TargetHeads`（`BlockCIDs`），并选择`BestTargetHead`从它
  - 步骤2.与最新观察到的头部同步，验证朝向它们的块（请求中间点）
  - 步骤3.随着验证的进行，`TargetHeads`并且`BestTargetHead`可能会改变，因为生产边缘的新块将到达，并且某些目标头或通往它们的路径可能无法验证。
  - 第4步。当节点“赶上”时完成`BestTargetHead`（检索所有状态，链接到本地链，验证所有块等）。
- 第5部分：保持同步，并参与块传播（`CHAIN_FOLLOW`）
  - 步骤1.如果安全条件发生变化，请返回至第4部分（`CHAIN_CATCHUP`）
  - 步骤2.接收，验证和传播收到的信息 `Blocks`
  - 第3步。现在更加确定拥有最佳链，最终确定“提示”并推进链状态。

`ChainSync`使用以下*概念性*状态机。由于这是一个*概念性的*状态机，因此实现可能会偏离精确实现这些状态或严格划分它们的状态。实现可能会模糊状态之间的界线。如果是这样，实现必须确保更改后的协议的安全性。

![image-20210201111716710](https://tva1.sinaimg.cn/large/008eGmZEgy1gn7v5rtlrlj31aw0petbh.jpg)

#### 2.4.3.4[同行发现](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync.peer-discovery)

对等发现是整个体系结构的关键部分。错误地操作可能会对协议的操作造成严重后果。当加入网络时，新节点最初连接的一组对等点可能会完全支配该节点对其他对等点的了解，因此，将主导该节点所拥有的网络状态。

对等发现可以通过任意外部手段来驱动，并被推到ChainSync所涉及协议（即GossipSub，Bitswap，BlockSync）的核心功能之外。这允许进行正交的，由应用程序驱动的开发，并且无需外部依赖来实现协议。但是，GossipSub协议支持：i）对等交换，以及ii）显式对等协议。

##### 2.4.3.4.1[同行交流](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync.peer-exchange)

Peer Exchange允许应用程序从一组已知的对等方进行引导，而无需外部对等方发现机制。此过程可以通过引导节点或其他普通对等节点来实现。**引导节点必须由系统操作员维护，并且必须正确配置。**它们必须稳定并且独立于协议构造（例如GossipSub网格构造）运行，也就是说，引导节点不维护与网格的连接。

有关Peer Exchange的更多详细信息，请参考 [GossipSub规范](https://github.com/libp2p/specs/tree/master/pubsub/gossipsub)。

##### 2.4.3.4.2[明确的对等协议](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync.explicit-peering-agreements)

使用明确的对等协议，运营商必须指定加入节点时节点应连接到的对等方列表。该协议必须具有可用于指定这些选项的选项。对于每个显式对等方，路由器必须建立并维持双向（对等）连接。

#### 2.4.3[ 渐进块验证](https://spec.filecoin.io/#section-systems.filecoin_blockchain.chainsync.progressive-block-validation)

- 为了使资源支出最小化，可以逐步进行[区块](https://spec.filecoin.io/#section-systems.filecoin_blockchain.struct.block)验证。
- 验证计算量很大，并且是严重的DOS攻击媒介。
- 安全的实现必须仔细安排验证时间，并在不完全验证块的情况下将修剪块的工作减至最少。
- `ChainSync`应该保留未验证块的缓存（最好按属于链的可能性排序），并在传递未验证块`FinalityTipset`或`ChainSync`承受大量资源时删除未验证块。
- 这些阶段可以部分地用于候选链中的许多块，以便在实际进行昂贵的验证工作之前很长时间就清除掉坏块。
- **块验证的渐进阶段**
  - **BV0-语法**：序列化，键入，值范围。
  - **BV1-合理的共识**：合理的矿工，重量和历时值（例如来自的链状状态`b.ChainEpoch - consensus.LookbackParameter`）。
  - **BV2-块签名**
  - **BV3-信标条目**：有效的随机信标条目已插入到块中（请参阅 [信标条目验证](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.validating-beacon-entries-on-block-reception)）。
  - **BV4-ElectionProof**：生成了有效的选举证明。
  - **BV5-WinningPoSt**：生成正确的PoSt。
  - **BV6-链血统和终结性**：验证区块链是否回到终结链，而不是终结性。
  - **BV7-消息签名**：
  - **BV8-状态树**：父提示消息执行将产生声明的状态树根和收据。

### 2.4.[储能共识](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus)

存储电源共识（SPC）子系统是使Filecoin节点能够就系统状态达成一致的主要接口。《存储功率共识》在其“[功率表”](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor.the-power-table)中考虑了各个存储矿工在给定链中超过共识的有效功率 。它还运行“ [预期共识”](https://spec.filecoin.io/#section-algorithms.expected_consensus)（Filecoin使用的基础共识算法），使存储矿工可以进行领导者选举并生成更新Filecoin系统状态的新块。

简而言之，SPC子系统提供以下服务：

- 访问每个子链的 [功率表](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor.the-power-table)，说明各个存储矿工的功率和链上的总功率。
- 访问 各个存储矿工的[预期共识](https://spec.filecoin.io/#section-algorithms.expected_consensus)，从而实现：
  - 根据协议的其余部分，访问[drand](https://spec.filecoin.io/#section-libraries.drand)提供的 可验证的随机性 [票证](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.tickets)。
  - 进行 [领导人选举](https://spec.filecoin.io/#section-algorithms.expected_consensus.secret-leader-election)产生新的障碍。
  - 使用EC的加权功能跨子链运行 [链选择](https://spec.filecoin.io/#section-algorithms.expected_consensus.chain-selection)。
  - 标识 [最近完成的提示集](https://spec.filecoin.io/#section-algorithms.expected_consensus.finality-in-ec)，供所有协议参与者使用。

#### 2.4.4.1[区分存储矿工和块矿工](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.distinguishing-between-storage-miners-and-block-miners)

在Filecoin网络中有两种获取Filecoin令牌的方法：

- 通过作为存储提供者参加 [存储市场](https://spec.filecoin.io/#section-systems.filecoin_markets.storage_market)并由客户支付文件存储交易的费用。
- 通过挖掘新的区块，扩展区块链，保护Filecoin共识机制以及运行智能合约以执行状态更新来作为 [Storage Miner](https://spec.filecoin.io/#section-systems.filecoin_mining.storage_mining)。

有两种类型的“矿工”（存储矿工和块矿工）可以区分。 Filecoin的[领导者选举](https://spec.filecoin.io/#section-algorithms.expected_consensus.secret-leader-election)取决于矿工的存储能力。因此，虽然所有区块矿工都将是存储矿工，但不一定相反。

但是，鉴于Filecoin的“有用的工作量证明”是通过文件存储（ [PoRep](https://spec.filecoin.io/#section-algorithms.pos.porep)和 [PoSt](https://spec.filecoin.io/#section-algorithms.pos.post)）实现的，存储矿工参与领导者选举的开销很少。这样的 [Storage Miner Actor](https://spec.filecoin.io/#section-systems.filecoin_mining.storage_mining.storage_miner_actor)仅需要向[Storage Power Actor](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor)注册 即可参与“预期共识”和矿区。

#### 2.4.4.2[上电](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.on-power)

质量调整后的功率作为其***扇区质量\***的静态函数分配给每个扇区，其中包括：i）**扇区时空**，它是扇区大小和承诺的存储持续时间的乘积，ii）**交易权重**，该权**重**转换由交易达成共识的权力，iii）**交易质量乘数**，该**乘数**取决于行业内完成的交易类型（即CC，常规交易或已验证的客户交易），最后，iv）**部门质量乘数**，即交易质量乘数乘以该行业中每种类型的交易所占用的时空量来加权。

该**部门的质量**是映射大小，持续时间和活跃交易的部门其一生的时间去对权力和报酬分配碰撞时类型的措施。

一个部门的质量取决于对该部门内部数据进行的交易。通常有三种类型的交易：*承诺容量（CC）*（实际上没有任何交易，而矿工在该部门内部存储任意数据），*常规交易*（矿工和客户就市场价格达成一致）以及在*验证客户*的交易，这给更多的权力部门。我们请读者阅读“ [扇区](https://spec.filecoin.io/#section-systems.filecoin_mining.sector)和 [扇区质量”](https://spec.filecoin.io/#section-systems.filecoin_mining.sector.sector_quality)部分，以获取有关扇区类型和扇区质量的详细信息，“ [已验证的客户”](https://spec.filecoin.io/#section-algorithms.verified_clients)部分可获得有关已验证的客户是什么的更多详细信息，以及 [CryptoEconomics](https://spec.filecoin.io/#section-algorithms.cryptoecon) 部分，以获取交易权重和质量乘数上的特定参数值。

**质量调整后的权力**是指矿工在“[秘密领导人选举”中](https://spec.filecoin.io/#section-algorithms.expected_consensus.secret-leader-election)所获得的票数，其 定义是随着矿工对网络的承诺存储量线性增加。

更准确地说，我们具有以下定义：

- *原始字节功率*：扇区大小，以字节为单位。
- *质量调整后的功率*：网络上存储的数据的共识功率，等于原始字节功率乘以扇区质量乘数。

#### 2.4.4.3[信标条目](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.beacon-entries)

Filecoin协议使用[drand](https://spec.filecoin.io/#section-libraries.drand)信标产生的 随机性来播种可在链中使用的无偏随机性种子（请参阅 [随机性](https://spec.filecoin.io/#section-algorithms.crypto.randomness)）。

反过来，这些随机种子由以下人员使用：

- 该 [sector_sealer](https://spec.filecoin.io/#section-systems.filecoin_mining.sector.sealing)为SealSeeds结合部门承诺给定的子链。
- 该 [post_generator](https://spec.filecoin.io/#section-systems.filecoin_mining.storage_proving.poster)为PoStChallenges证明部门继续致力于为给定块的。
- Storage Power子系统作为“[秘密领导者”选举中的](https://spec.filecoin.io/#section-algorithms.expected_consensus.secret-leader-election)随机性， 用于确定选择矿工采矿新区块的频率。

该随机性可以通过根据其安全性要求使用它们的各个协议从各种Filecoin链纪元中得出。

重要的是要注意，给定的Filecoin网络和给定的drand网络不必具有相同的循环时间，即Filecoin生成的块可能比drand生成的随机性更快或更慢。例如，如果drand信标产生的随机性是Filecoin产生块的两倍，那么我们可能期望在Filecoin时代产生两个随机值，反之，如果Filecoin网络的速度是drand的两倍，则我们可能期望一个随机值每隔一个Filecoin时代。因此，取决于两个网络的配置，某些Filecoin块可能包含多个或不包含drand条目。此外，必须确保中断期间对drand网络进行的新的随机性条目的任何呼叫都应被阻止，如`drand.Public()` 下面的电话。在所有情况下，Filecoin块都必须包括自`BeaconEntries`块头字段中的最后一个纪元以来生成的所有drand beacon输出。给定Filecoin纪元的任何随机性使用都应使用Filecoin块中包含的最后一个有效drand条目。如下所示。

##### 2.4.4.3.1[获取VM的drand随机性](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.get-drand-randomness-for-vm)

对于诸如PoRep创建，证明验证之类的操作，或者需要Filecoin VM随机性的任何操作，应该有一种方法可以从链中正确提取drand条目。请注意，如果drand较慢，则该回合可能跨越多个filecoin时期。最低的时期号块将包含请求的信标条目。同样，如果在应插入信标的位置存在零轮，我们需要在链上进行迭代以找到将条目插入到的位置。具体而言，下一个非空块必须包含定义所请求的drand条目。

##### 2.4.4.3.2[从drand网络获取随机性](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.fetch-randomness-from-drand-network)

进行挖掘时，矿工可以从drand网络中获取条目，以将其包括在新块中。

[示例：DrandBeacon](https://spec.filecoin.io/#example-drandbeacon)

DrandBeacon将Lotus与drand网络相连，以便以与Filecoin轮次/纪元一致的方式向系统提供随机性。

我们通过其公共HTTP端点连接到drand对等方。对等点在drandServers变量中枚举。

Drand链的根信任是从build.DrandChain配置的。

```go
type DrandBeacon struct {
	client dclient.Client

	pubkey kyber.Point

	// seconds
	interval time.Duration

	drandGenTime uint64
	filGenTime   uint64
	filRoundTime uint64

	cacheLk    sync.Mutex
	localCache map[uint64]types.BeaconEntry
}
```

[示例：BeaconEntriesForBlock](https://spec.filecoin.io/#example-beaconentriesforblock)

```go
func BeaconEntriesForBlock(ctx context.Context, bSchedule Schedule, epoch abi.ChainEpoch, parentEpoch abi.ChainEpoch, prev types.BeaconEntry) ([]types.BeaconEntry, error) {
	{
		parentBeacon := bSchedule.BeaconForEpoch(parentEpoch)
		currBeacon := bSchedule.BeaconForEpoch(epoch)
		if parentBeacon != currBeacon {
			// Fork logic
			round := currBeacon.MaxBeaconRoundForEpoch(epoch)
			out := make([]types.BeaconEntry, 2)
			rch := currBeacon.Entry(ctx, round-1)
			res := <-rch
			if res.Err != nil {
				return nil, xerrors.Errorf("getting entry %d returned error: %w", round-1, res.Err)
			}
			out[0] = res.Entry
			rch = currBeacon.Entry(ctx, round)
			res = <-rch
			if res.Err != nil {
				return nil, xerrors.Errorf("getting entry %d returned error: %w", round, res.Err)
			}
			out[1] = res.Entry
			return out, nil
		}
	}

	beacon := bSchedule.BeaconForEpoch(epoch)

	start := build.Clock.Now()

	maxRound := beacon.MaxBeaconRoundForEpoch(epoch)
	if maxRound == prev.Round {
		return nil, nil
	}

	// TODO: this is a sketchy way to handle the genesis block not having a beacon entry
	if prev.Round == 0 {
		prev.Round = maxRound - 1
	}

	cur := maxRound
	var out []types.BeaconEntry
	for cur > prev.Round {
		rch := beacon.Entry(ctx, cur)
		select {
		case resp := <-rch:
			if resp.Err != nil {
				return nil, xerrors.Errorf("beacon entry request returned error: %w", resp.Err)
			}

			out = append(out, resp.Entry)
			cur = resp.Entry.Round - 1
		case <-ctx.Done():
			return nil, xerrors.Errorf("context timed out waiting on beacon entry to come back for epoch %d: %w", epoch, ctx.Err())
		}
	}

	log.Debugw("fetching beacon entries", "took", build.Clock.Since(start), "numEntries", len(out))
	reverse(out)
	return out, nil
}
```

[示例：MaxBeaconRoundForEpoch](https://spec.filecoin.io/#example-maxbeaconroundforepoch)

```go
func (db *DrandBeacon) MaxBeaconRoundForEpoch(filEpoch abi.ChainEpoch) uint64 {
	// TODO: sometimes the genesis time for filecoin is zero and this goes negative
	latestTs := ((uint64(filEpoch) * db.filRoundTime) + db.filGenTime) - db.filRoundTime
	dround := (latestTs - db.drandGenTime) / uint64(db.interval.Seconds())
	return dround
}
```

##### 2.4.4.3.3[在块接收时验证信标条目](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.validating-beacon-entries-on-block-reception)

Filecoin链将包含从Filecoin起源到当前区块的整个信标输出。

考虑到它们在领导者选举和Filecoin中其他关键协议中的作用，必须为每个区块验证区块的信标条目。有关详细信息，请参见 [drand](https://spec.filecoin.io/#section-libraries.drand)。可以通过使用drand的[`Verify`](https://github.com/drand/drand/blob/763e9a252cf59060c675ced0562e8eba506971c1/chain/beacon.go#L76)端点，确保每个信标条目都是链中前一个条目的有效签名来完成此操作 ：

[示例：ValidateBlockValues](https://spec.filecoin.io/#example-validateblockvalues)

```go
func ValidateBlockValues(bSchedule Schedule, h *types.BlockHeader, parentEpoch abi.ChainEpoch,
	prevEntry types.BeaconEntry) error {
	{
		parentBeacon := bSchedule.BeaconForEpoch(parentEpoch)
		currBeacon := bSchedule.BeaconForEpoch(h.Height)
		if parentBeacon != currBeacon {
			if len(h.BeaconEntries) != 2 {
				return xerrors.Errorf("expected two beacon entries at beacon fork, got %d", len(h.BeaconEntries))
			}
			err := currBeacon.VerifyEntry(h.BeaconEntries[1], h.BeaconEntries[0])
			if err != nil {
				return xerrors.Errorf("beacon at fork point invalid: (%v, %v): %w",
					h.BeaconEntries[1], h.BeaconEntries[0], err)
			}
			return nil
		}
	}

	// TODO: fork logic
	b := bSchedule.BeaconForEpoch(h.Height)
	maxRound := b.MaxBeaconRoundForEpoch(h.Height)
	if maxRound == prevEntry.Round {
		if len(h.BeaconEntries) != 0 {
			return xerrors.Errorf("expected not to have any beacon entries in this block, got %d", len(h.BeaconEntries))
		}
		return nil
	}

	if len(h.BeaconEntries) == 0 {
		return xerrors.Errorf("expected to have beacon entries in this block, but didn't find any")
	}

	last := h.BeaconEntries[len(h.BeaconEntries)-1]
	if last.Round != maxRound {
		return xerrors.Errorf("expected final beacon entry in block to be at round %d, got %d", maxRound, last.Round)
	}

	for i, e := range h.BeaconEntries {
		if err := b.VerifyEntry(e, prevEntry); err != nil {
			return xerrors.Errorf("beacon entry %d (%d - %x (%d)) was invalid: %w", i, e.Round, e.Data, len(e.Data), err)
		}
		prevEntry = e
	}

	return nil
}
```

#### 2.4.4.4[门票](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.tickets)

Filecoin块标题还包含一个从其时代的信标条目生成的“票”。对于等重的货叉，票证用于在“货叉选择规则”中打破平局。

每当在Filecoin中比较票证时，比较就是票证的VRF摘要的字节。

##### 2.4.4.4.1[随机票生成](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.randomness-ticket-generation)

在Filecoin时代`n`，将使用相应的信标条目生成新的票证`n`。

矿工通过可验证随机函数（VRF）运行信标条目以获取新的唯一票证。信标条目前面带有票证域分隔标签，并与矿工参与者地址相连（以确保使用相同工作人员密钥的矿工获得不同的票证）。

生成给定纪元n的票证：

```text
randSeed = GetRandomnessFromBeacon(n)
newTicketRandomness = VRF_miner(H(TicketProdDST || index || Serialization(randSeed, minerActorAddress)))
```

[可验证的随机函数](https://spec.filecoin.io/#section-algorithms.crypto.vrf)用于票证生成。

##### 2.4.4.4.2[票证验证](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.ticket-validation)

每个票证应从VRF链中的前一个票证生成，并进行相应的验证。

#### 2.4.4.5[最小矿工规模](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.minimum-miner-size)

为了确保存储电源共识，系统定义了参与共识所需的最小矿机大小。

具体而言，矿工必须至少`MIN_MINER_SIZE_STOR`具有电量（即当前在存储交易中使用的存储电源）才能参与领导者选举。如果没有矿工拥有`MIN_MINER_SIZE_STOR`或拥有更多权力，则至少具有与矿工顶部最小`MIN_MINER_SIZE_TARG`的矿工（按存储功率排序）的功率相同的矿工将能够参加领导人选举。以简单的英语`MIN_MINER_SIZE_TARG = 3`为例，这意味着具有至少与第三大矿商相同权力的矿工将有资格参加共识。

小于此值的矿工无法在网络中进行区块挖掘并获得区块奖励。即使他们的权力不会被算作领导人选举的票数，他们的权力仍将计入整个网络（原始或要求的）存储能力中。但是，**重要的是要注意，此类矿工仍然会遭受其权力的过失并因此受到惩罚**。

因此，要引导网络，起源块必须包括矿工（可能只是CommittedCapacity扇区）来启动网络。

该`MIN_MINER_SIZE_TARG`条件将不会在任何矿工拥有的`MIN_MINER_SIZE_STOR`权力都超过的网络中使用。尽管如此，它的定义是为了确保小型网络中的活动（例如，接近起源或在掉电之后）。

#### 2.4.4.6[储能演员](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor)

##### 2.4.4.6.1[`StoragePowerActorState` 实作](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor.storagepoweractorstate-implementation)

[示例：状态](https://spec.filecoin.io/#example-state)

```go
type State struct {
	TotalRawBytePower abi.StoragePower
	// TotalBytesCommitted includes claims from miners below min power threshold
	TotalBytesCommitted  abi.StoragePower
	TotalQualityAdjPower abi.StoragePower
	// TotalQABytesCommitted includes claims from miners below min power threshold
	TotalQABytesCommitted abi.StoragePower
	TotalPledgeCollateral abi.TokenAmount

	// These fields are set once per epoch in the previous cron tick and used
	// for consistent values across a single epoch's state transition.
	ThisEpochRawBytePower     abi.StoragePower
	ThisEpochQualityAdjPower  abi.StoragePower
	ThisEpochPledgeCollateral abi.TokenAmount
	ThisEpochQAPowerSmoothed  smoothing.FilterEstimate

	MinerCount int64
	// Number of miners having proven the minimum consensus power.
	MinerAboveMinPowerCount int64

	// A queue of events to be triggered by cron, indexed by epoch.
	CronEventQueue cid.Cid // Multimap, (HAMT[ChainEpoch]AMT[CronEvent])

	// First epoch in which a cron task may be stored.
	// Cron will iterate every epoch between this and the current epoch inclusively to find tasks to execute.
	FirstCronEpoch abi.ChainEpoch

	// Claimed power for each miner.
	Claims cid.Cid // Map, HAMT[address]Claim

	ProofValidationBatch *cid.Cid // Multimap, (HAMT[Address]AMT[SealVerifyInfo])
}
```

##### 2.4.4.6.2[`StoragePowerActor` 实作](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor.storagepoweractor-implementation)

[示例：出口](https://spec.filecoin.io/#example-exports)

```go
func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.CreateMiner,
		3:                         a.UpdateClaimedPower,
		4:                         a.EnrollCronEvent,
		5:                         a.OnEpochTickEnd,
		6:                         a.UpdatePledgeTotal,
		7:                         nil, // deprecated
		8:                         a.SubmitPoRepForBulkVerify,
		9:                         a.CurrentTotalPower,
	}
}
```

[示例：MinerConstructorParams](https://spec.filecoin.io/#example-minerconstructorparams)

此处定义了存储矿工参与者的构造函数参数，以便高级参与者可以将它们发送给初始化参与者以实例化矿工。从v0开始更改：

- 添加了ControlAddrs

```go
type MinerConstructorParams struct {
	OwnerAddr     addr.Address
	WorkerAddr    addr.Address
	ControlAddrs  []addr.Address
	SealProofType abi.RegisteredSealProof
	PeerId        abi.PeerID
	Multiaddrs    []abi.Multiaddrs
}
```

##### 2.4.4.6.3[功率表](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor.the-power-table)

给定矿工通过EC的领导者选举产生的区块部分（因此他们获得的区块奖励）与他们`Quality-Adjusted Power Fraction`的时间成正比。也就是说，质量调整后的功率代表网络上总质量调整后的功率的1％的矿工应按预期开采1％的区块。

SPC提供了一个功率表抽象，可随时间跟踪矿工功率（即矿工存储相对于网络存储）。针对新的部门承诺（增加矿工功率），失败的PoSt（减少矿工功率）或其他存储和共识故障，更新了功率表。

部门ProveCommit是第一次向网络证明功率，因此，成功的部门ProveCommit会首先添加功率。当宣布一个扇区已恢复时，也会添加电源。矿工有望在其贡献力量的所有领域进行证明。

当扇区到期，声明或检测到扇区有故障或通过矿工调用终止时，功率会降低。矿工还可以通过延长部门的寿命`ExtendSectorExpiration`。

功率表中的Miner生命周期应大致如下：

- `MinerRegistration`：存储挖掘子系统将一个具有关联工作人员公用密钥和地址的新矿工及其关联的扇区大小（在每个工作人员中只有一个）注册到了功率表中。

- ```
  UpdatePower
  ```

  ：这些功率增量和减量由各种存储参与者调用（因此必须由网络上的每个完整节点进行验证）。特别：

  - 功率增加为 `SectorProveCommit`
  - 丢失WindowPoSt（`DetectedFault`）后，分区的功效会立即降低。
  - 当特定扇区通过“声明的故障”或“跳过的故障”进入故障状态时，其功率将减小。
  - PoSt宣布并证明恢复后，将增加特定部门的力量。
  - 通过矿工发票到期或终止某个特定部门的电源后，该电源将被删除。

总而言之，只有处于活动状态的扇区才能控制电源。在上添加扇区后，该扇区将变为活动状态`ProveCommit`。进入故障状态后，电源立即减小。确认已声明恢复后，电源即会恢复。通过矿工调用终止或终止某个扇区的电源后，该电源将被删除。

##### 2.4.4.6.4[质押抵押品](https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor.pledge-collateral)

质押抵押品因影响存储电源共识的任何故障而被削减，其中包括：

- 尤其是预期的共识错误（请参阅“ [共识错误”](https://spec.filecoin.io/#section-algorithms.expected_consensus.consensus-faults)），砍杀者会将这些错误报告给，`StoragePowerActor`以换取奖励。
- 影响共识功率的故障，更一般而言，尤其是未承诺的功率故障（即， [存储故障](https://spec.filecoin.io/#section-systems.filecoin_markets.onchain_storage_market.faults.storage-faults)），将由`CronActor`自动报告，或者当矿工终止扇区的时间早于承诺的持续时间时，将报告这些[故障](https://spec.filecoin.io/#section-systems.filecoin_markets.onchain_storage_market.faults.storage-faults)。

有关质押抵押品的更详细讨论，请参阅 [矿工抵押品部分](https://spec.filecoin.io/#section-systems.filecoin_mining.miner_collaterals)。







