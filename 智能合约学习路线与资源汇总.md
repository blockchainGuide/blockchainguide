# 以太坊学习路线和资源汇总

笔者相对擅长合约安全方面，因此这个学习路线大致是**偏向于智能合约开发和智能合约安全**，对于很多从事开发的朋友，可能显得比较学院派，不是那么切合工作实际，不过抛砖引玉，欢迎讨论和补充。学习的资源可以在下方的资源汇总中找到，笔者日后将会写合约审计方面的文章。

我们学习的心得、理论基础和源码分析，都会写在仓库里，欢迎交流学习

https://github.com/learnerLj/geth-analyze

这下面的内容笔者也没有完全掌握，但是会逐渐的学习，在未来 3-5 年研究生毕业后也许能够在合约安全、安全的区块链系统构建等方面有一定的成就。

## 第一步：完成简单 DApp 开发

一开始入门就要求做简单的 DApp 可能看起来不合理，因为读者可能现在都不知道 DApp 是什么。但是项目驱动的学习将会非常有效，并且掌握的开发技能将会在后续的学习中发挥重要作用。

将会学会的知识有：

1. **编程语言：JavaScript, Solidity, (HTML, CSS)** 

完成前端与合约交互往往用的 JavaScript 的 API，这是必会的技能。JavaScript 也不必学的多深入，能够熟练的掌握 promis/async 之类的异步操作，编写函数和类即可。学习 JavaScript 对于编写合约的测试用例和一些重复工作的自动化非常有帮助。每次实验中不可能重复部署实验环境，而是一次编写，反复实验和调整。

Solidity 是编写智能合约最主流的语言，也将会是只能合约开发和智能合约安全的基础，是我们后续需要精通的语言。

至于 HTML 和 CSS 本身不难，和区块链关系不大，视自身情况学习，可以弱化。

2. **Linux 的熟练使用**

Linux 是以太坊客户端运行的主要系统环境，因为它相对于 windows 有更完善的命令行工具，过去的许多教程也是要么基于 MacOS，要么基于 Linux。这里推荐使用 Ubuntu 20.04 长期支持版，这是用的最多的带图形交互页面的 Linux 系统。

3. **区块链入门知识**

市面上关于区块链的书多如牛毛，笔者在刚入门时几乎借助 掌阅、微信读书等电子书平台和学校图书馆，读遍了所有中文书籍。可惜的是，笔者发现它们的内容稂莠不齐，甚至怀疑写书的人是否真的理解他所表达的内容，要么毫无洞见，要么写成教材式的死记硬背概念，少有佳作。笔者选出几本觉得还可以的书作为入门的阅读材料。

《精通以太坊》、《区块链架构之美——从比特币、以太坊、超级账本看区块链架构设计》《深入理解以太坊》是可以仔细阅读的，部分内容读不懂也没关系，可以会过来再读。

《深入浅出区块链核心技术与项目分析》、《深入以太坊智能合约开发》《区块链：以太坊 DApp 开发实战》值得略读，浏览内容，了解区块链的方方面面。其中《区块链：以太坊 DApp 开发实战》的中继服务器开发，值得以后掌握 Go 语言和具有一定的源码基础后尝试实践。

除此之外，可以[阅读笔者其他的文章](https://www.zhihu.com/people/luo-jia-hao-34-25/columns)，会持续更新。也可以浏览下面的资源汇总，大致的看一遍，建立对区块链的模糊认识。

4. **开发框架和库**

实现 DApp 时交易使用 truffle 或者 hardhat 这样的集成开发框架。truffle 主要使用 web3.js 而 hardhat 主要使用 ether.js，这两个都可以，但是后者最近较为流行。

除了 web3.js 和 ether.js 这样的用 JavaScript 编写的完整的库外，还有其他语言编写的库，如 web3j、web3.py，但是建议和所使用的框架配套。

## 第二部分：深入合约开发和合约安全

通过第一部分的学习，我们可以假设读者

- [ ] 能够在 Linux 命令行熟练使用框架.
- [ ] 熟悉区块链的基本概念，包括区块、合约、账户、交易池、区块生成过程、交易的核心数据部分（如 `value`、`data`)、`receipt`、`gas`、基本了解 PoS 和PoW 、最长链原则
- [ ] 掌握编写合约的主要知识，例如 `receive` 函数 和 `fallback` 函数，编写合约的测试脚本。
- [ ] 能够使用 geth 这样的客户端，搭建基本的网络
- [ ] 知道在区块浏览器上查询信息

**接下来开始了解区块链的相对高级的主体，注意偏向合约开发和合约安全方向：**

- [ ] 了解 ABI 的编码方式，并且能够使用 Solidity 内置的 ABI 相关函数。
- [ ] 了解函数签名的生成方式以及在合约执行时的作用。
- [ ] 了解预言机的确切含义，以及在链上的实现方式。
- [ ] 理解以太坊虚拟机
  - [ ] 了解交易发送到调用合约的整个过程。
  - [ ] 熟悉合约中的数据在存储中的组织形式、内存的组织形式、`calldata` 的组织形式。
  - [ ] 初步理解 Solidity 编译器的原理，能够读懂汇编
- [ ] 了解以太坊的字节码、操作码，能供单步调试，观察栈、内存的变化。
- [ ] 了解合约的内联汇编，直接操作数据。
- [ ] 根据自己的实际要求，选择是否需要熟悉各种业务代码，例如代币等等

**然后，阅读以太坊的源码，建议阅读 geth，因为它时最主流也是最活跃的区块链项目。**

对于合约开发方向，深入底层似乎有些有些不必要了，但是对于合约安全来说，接下来的内容可能才算开始“入门”。

- [ ] 理解以太坊最重要的数据组织形式：状态树、收据树、交易树
  - [ ] 掌握 Trie Tree、Patrica Tree、Merkle Tree 的数据结构基础。
  - [ ] 掌握重要的编码方式： HP 编码、 RLP 编码。
- [ ] 看懂源码中的 core 部分，从源码实现层次
  - [ ] 理解区块链中的数据结构的确切定义和实现的方法。例如，从代码层次，掌握 `block`、`transction`、`log`、`receipt` 的定义和对应的方法。
  - [ ] 理解创世块的的生成方式
  - [ ] 看懂交易池（**非常重要**），例如交易排序、并发执行、交易广播实时更新数据
  - [ ] 看懂交易广播、区块形成过程等等
- [ ] 掌握合约的常见漏洞和审计工具的使用

## 第三部分：高级智能合约安全研究

- [ ] 深入理解字节码，合约存储、链上和链下数据完整性和安全性方案
- [ ] 掌握合约安全分析的方法，理解原理，包括
  - [ ] 用机器学习方法分析
  - [ ] 符号执行方法
  - [ ] 模糊测试分析
  - [ ] 污点分析
  - [ ] 形式验证分析
- [ ] 掌握区块链系统态势感知模型，能够设计系统方案，动态、整体地监控链上数据，并且自动分析可能的安全威胁。

# 资源汇总

这些资源主要来自笔者的经历和学习，也参考了许多人的总结，作为自己的系统作结，应该对大家有帮助。

### 帮助和交流平台

- [gitter](https://gitter.im/ethereum/home) 在线频道。
- [reddit ethereum](https://www.reddit.com/r/ethereum/) 讨论区
- 以太坊的 [stackexchange](https://ethereum.stackexchange.com/)
- [stackoverflow](https://stackoverflow.com/)

还有一些 discord 服务器，感兴趣可以加入。 

### 优质社区

- [以太坊社区网络](https://www.ethereum.cn/)，他们的文章整理的不错，[文档](https://www.ethereum.cn/develop)也很好。

![image-20220311181349565](https://gitee.com/learnerLj/typora/raw/master/202203111813924.png)

- [以太坊维基百科](https://eth.wiki/)，可以浏览基本的概念，很有帮助。

- [以太坊基金会博客](https://blog.ethereum.org/)，可以得到很多前沿信息。

![image-20220312003259327](https://gitee.com/learnerLj/typora/raw/master/202203120032607.png)

- [登链社区](https://learnblockchain.cn/)，许多翻译的文章质量很高，并且有一些文档翻译。
- [以太坊知识库](https://learnblockchain.cn/eth/)，虽然许多东西没有更新，但是有些文章翻译的很好，例如 [新手入门](https://learnblockchain.cn/eth/basic.html)、以及它开发者提供的[参考](https://learnblockchain.cn/eth/dev.html) 都可以看出来是花了很长时间整理的，可以浏览，增长见识。比较可惜的是，后续没有继续更新，可能是小白成为大佬的过程常常写很多的文章，但是成为大佬后都应对复杂问题，就没有继续写下去了吧。

![image-20220311184548871](https://gitee.com/learnerLj/typora/raw/master/202203111845175.png)

- [layer2 方案的备忘录](https://mirror.xyz/ethmaxitard.eth/iyCAlOexgQKOvoSAAk4utYGEdnESOKb5HstM2_LaqL4)，以后深入以太坊拓展方案后可以很快对 layer2 有大致的认识。
- [medium](https://medium.com/) 虽然是英文社区，但是里面的优秀文章写的很好，建议翻译慢慢看。

![image-20220311184518417](https://gitee.com/learnerLj/typora/raw/master/202203111845710.png)

- [Eth2 展望和分享](https://hackmd.io/@benjaminion)，作者经常分享自己对以太坊升级的看法和资讯。

![image-20220311185005411](https://gitee.com/learnerLj/typora/raw/master/202203111850716.png)

### 入门参考

- [《以太坊的指南针》 ‒ 以太坊的指南针 1.0.0 documentation](https://ethbook.abyteahead.com/index.html) 面向 **没接触过以太坊的人** 的书。
- 以太坊的入门级[简单文档](https://www.kancloud.cn/qq393855529/ethereum/837511)
- [区块链概念和基础](https://zhuanlan.zhihu.com/p/331906800)（基于比特币）
- [官方文档](https://ethereum.org/zh/developers/docs/)，内容可能比较简洁，相对深入，可以量力而行。
- 《[精通以太坊](https://github.com/inoutcode/ethereum_book)》前面的内容适合入门，后面的内容需要一定的基础，可日后再回顾。开源书，有中文版。

![Mastering Ethereum](https://github.com/inoutcode/ethereum_book/raw/master/images/cover.png)





### 智能合约合约教程

- [合约编写示例](https://solidity-by-example.org/)

- [笔者的笔记](https://zhuanlan.zhihu.com/p/459969916)

- [中文文档](https://learnblockchain.cn/docs/solidity/index.html)

- [编游戏的同时学习以太坊 DApp 开发](https://cryptozombies.io/zh/)

  虽然这个教程比较老，但是比较有趣![image-20220312003347907](https://gitee.com/learnerLj/typora/raw/master/202203120033140.png)

- [介绍 Solidity 细节的视频](https://www.youtube.com/channel/UCJWh7F3AFyQ_x01VKzr9eyA)（在Youtube上）

- [系统性的合约开发教程](https://www.youtube.com/c/%E7%90%86%E6%83%B3%E5%8C%BA%E5%9D%97%E9%93%BE%E5%AD%A6%E9%99%A2) （在Youtube上）

### 区块浏览器

- [Etherscan](https://etherscan.io/) – *中文、韩文、俄文和日文*
- [Etherchain](https://www.etherchain.org/)
- [Ethplorer](https://ethplorer.io/) - *支持中文、西班牙语、法语、土耳其语和俄语*
- [Blockchair](https://blockchair.com/ethereum) --*也有西班牙文、法文、意大利文、荷兰文、葡萄牙文、俄文、中文和波斯文*
- [Blockscout--区块链浏览器](https://blockscout.com/)
- [OKLink](https://www.oklink.com/eth)
- https://beaconcha.in/
- https://beaconscan.com/
- [https://eth2stats.io/](https://eth2stats.io/medalla-testnet)
- https://ethscan.org/（beaconcha.in 的分叉）

一些特殊的浏览器

- [统计被销毁的 ETH](https://watchtheburn.com/)
- [节点信息和链状态概览](https://ethstats.net/)。

### 合约库

- [openzeppelin/contracts](openzeppelin/contracts) 提供了常见的合约库，实现了一些标准库，如 ERC20、ERC721，仓库中可阅读[源码](https://github.com/OpenZeppelin/openzeppelin-contracts)。

![image-20220311114749585](https://gitee.com/learnerLj/typora/raw/master/image-20220311114749585.png)

- [dappsys](https://github.com/dapphub/dappsys) 是一个新兴的合约库，用法参见文档[文档](https://dappsys.readthedocs.io/en/latest/)

![image-20220311114632238](https://gitee.com/learnerLj/typora/raw/master/image-20220311114632238.png)

- [HQ20](https://github.com/HQ20/contracts) 里提供里许多的合约和他们的测试脚本，可以作为自己开发的组件。 

### 集成开发工具

**[Truffle+Ganache](https://trufflesuite.com/index.html)**

使用开发环境可以加快重复性任务，例如 Truffle 主要完成以下工作：

- 编制合同
- 部署合约
- 调试合约
- 升级合约
- 运行单元测试

![松露框架](https://yos.io/assets/contract1.png)

Ganache 最近升级后功能强大了许多，可以提供本地私链的测试环境，而且还支持其他功能。

![您可以在本地运行 Ganache 以加快开发速度。](https://yos.io/assets/contract3.png)

其次，truffle 有很多的插件，

- [truffle-plugin-verify](https://github.com/rkalis/truffle-plugin-verify) - 验证 Etherscan 上的特定地址的智能合约代码是否和本地相同
- [truffle -security](https://github.com/ConsenSys/truffle-security) - 对智能合约运行 MythX 安全分析。
- [truffle-contract-size](https://github.com/IoBuilders/truffle-contract-size) - 以千字节显示智能合约的大小。
- [truffle-plugin-abigen](https://github.com/ChainSafe/truffle-plugin-abigen) - 生成与 Geth 兼容的[abigen](https://github.com/ethereum/go-ethereum/wiki/Native-DApps:-Go-bindings-to-Ethereum-contracts)数据，用于为以太坊合约构建 Golang 绑定
- [openzeppelin-upgrades](https://github.com/OpenZeppelin/openzeppelin-upgrades) 可升级合约插件
- [solidity-coverage](https://github.com/sc-forks/solidity-coverage) 检查测试的覆盖性。

官网中还介绍了其他的开发框架：

**Hardhat** 

hardhat 的堆栈跟踪功能很强大，报错信息相对更具有可读性，而且主网分叉功能也很赞。

![image-20220311153827948](https://gitee.com/learnerLj/typora/raw/master/202203111538382.png)

- [hardhat.org](https://hardhat.org/)，除了官网也有国人不完全的[翻译](https://learnblockchain.cn/docs/hardhat/getting-started/)。
- [GitHub](https://github.com/nomiclabs/hardhat)

**Brownie - **基于 Python 的开发环境和测试框架。

- [相关文档](https://eth-brownie.readthedocs.io/en/latest/)
- [GitHub](https://github.com/eth-brownie/brownie)

**Embark -** 开发环境、测试框架以及与以太坊、IPFS 和 Whisper 集成的其他工具。

![Embark](https://github.com/embarklabs/embark/raw/master/header.jpg)

- [相关文档](https://embark.status.im/docs/)
- [GitHub](https://github.com/embark-framework/embark)

**Epirus -** 在 JVM 上开发区块链应用程序的平台。

- [主页](https://www.web3labs.com/web3j-sdk)
- [相关文档](https://docs.web3j.io/)
- [GitHub](https://github.com/web3j/web3j)

**OpenZeppelin SDK -** 工具包：一套帮助您开发、编译、升级、部署智能合约并与之交互的工具。

- [OpenZepelin SDK](https://openzeppelin.com/sdk/)
- [GitHub](https://github.com/OpenZeppelin/openzeppelin-sdk)
- [社区论坛](https://forum.openzeppelin.com/c/support/17)

- https://github.com/PaulRBerg/create-eth-app/tree/develop/templates)

**The Graph -** 链上信息查询的API

- [网站](https://thegraph.com/)
- [使用教程](https://ethereum.org/zh/developers/tutorials/the-graph-fixing-web3-data-querying/)

**Alchemy -** layer2 平台提供的开发套件。

- [alchemy.com](https://www.alchemy.com/)
- [GitHub](https://github.com/alchemyplatform)
- [Discord](https://discord.com/invite/A39JVCM)

**Dapptools -** 一套专注于以太坊的 CLI 工具，遵循 Unix 设计理念，倾向于可组合、可配置和可扩展性。

- [主页](https://dapp.tools/)
- [GitHub](https://github.com/dapphub/dapptools/)

![image-20220311172321074](https://gitee.com/learnerLj/typora/raw/master/202203111723295.png)

看官网朴实的介绍，确实很符合 unix 极简主义的哲学

关于其他集成开发工具的介绍，可以见 [2022十大智能合约开发工具](https://zhuanlan.zhihu.com/p/459165804) 

### 与合约交互的库

**Web3.js -** 是使用的非常广泛的库，包含完整的合约交互和查询函数

- [最新英文文档](https://web3js.readthedocs.io/en/1.0/)，也有翻译的[中文文档](https://learnblockchain.cn/docs/web3.js/index.html)。web3.js 经过了 0.2 到 1.0 的大版本更新，使用时需要注意用法的不同。
- [GitHub](https://github.com/ethereum/web3.js/)

**Ethers.js -** 现在的主流库，hardhat 框架也主要用到它。

- [文档](https://docs.ethers.io/)
- [GitHub](https://github.com/ethers-io/ethers.js/)

**Graph -** 比较新的使用 GraphQL 的信息查询的库，配套有信息查询平台。

- [Graph](https://thegraph.com/)
- [Graph Explorer](https://thegraph.com/explorer/)
- [相关文档](https://thegraph.com/docs/)
- [GitHub](https://github.com/graphprotocol/)

**Alchemyweb3 -** 封装后的 Web3.js 的库，Alchemy 的配套生态

- [相关文档](https://docs.alchemyapi.io/documentation/alchemy-web3)
- [GitHub](https://github.com/alchemyplatform/alchemy-web3)

**Web3j** - Java 的API

- [GitHub](https://github.com/web3j/web3j)

![image-20220311154449037](https://gitee.com/learnerLj/typora/raw/master/202203111544404.png)

**Web3.py** - 用于与以太坊交互的 Python 库

- [GitHub](https://github.com/ethereum/web3.py)
- [文档](https://web3py.readthedocs.io/en/stable/)

### DApp 的前端库

前端链接钱包往往需要阅读不同的钱包的文档，然后针对性的写代码，这是重复性的工作，因此有相应的库完成了这个工作。

- [blocknative](https://www.blocknative.com/) 不只提供了连接钱包的接口，而且还提供了监控内存池、交易池的接口。

![image-20220311153731102](https://gitee.com/learnerLj/typora/raw/master/202203111537392.png)

- [web3modal](https://github.com/Web3Modal/web3modal) 提供链接钱包的 UI 和封装。

![image-20220312175944659](https://gitee.com/learnerLj/typora/raw/master/202203121759262.png)

- [web3-react](https://github.com/NoahZinsmeister/web3-react) 为前端 React 提供的 DAPP 开发组件。

- **创建 Eth App -** 一大量的创建 DApp 的模板

  - [GitHub](https://github.com/paulrberg/create-eth-app)

- **Scaffold-Eth -** Ethers.js + Hardhat + React 的 DApp 模板

  - [GitHub](https://github.com/austintgriffith/scaffold-eth)

  ![image](https://user-images.githubusercontent.com/2653167/124158108-c14ca380-da56-11eb-967e-69cde37ca8eb.png)

- [Drizzle](https://trufflesuite.com/drizzle/) 擅长大量状态管理的前端库。

  ![image-20220311180840995](https://gitee.com/learnerLj/typora/raw/master/202203111808462.png)



### 安全审计

- [Mythril](https://github.com/ConsenSys/mythril) 是 EVM 字节码的安全分析工具。它使用符号执行、SMT 解决和污点分析来检测各种安全漏洞。
- [Slither](https://github.com/crytic/slither) 是一个用 Python 3 编写的 Solidity 静态分析框架。
- [Manticore](https://github.com/trailofbits/manticore) 是用于分析智能合约和二进制文件的符号执行工具。
- [Echidna](https://github.com/crytic/echidna) 是一个 Haskell 程序，旨在对以太坊智能合约进行模糊测试/基于属性的测试。
- [tenderly](https://dashboard.tenderly.co/learnerL/project/forks) 是集成、开发、测试模拟的平台，主网分叉的功能很赞。

这方面的内容，我会在后续合约审计文章中详细说明各种工具的使用和原理。

### 漏洞分析博客

- [慢雾科技的安全技术探究](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU4ODQ3NTM2OA==&action=getalbum&album_id=1378653641065857025&scene=173&from_msgid=2247494336&from_itemidx=1&count=3&nolastread=1#wechat_redirect) 里面会分享漏洞分析的报告。
- [合约漏洞赏金平台 immunefi](https://immunefi.com/)，在上面提交漏洞报告，不仅可以得到丰厚的回报，也会收获行业声誉，也提供了[智能合约安全的教程](https://immunefi.com/learn/)。

![image-20220312180143628](https://gitee.com/learnerLj/typora/raw/master/202203121801287.png)

- [rekt](https://rekt.news/zh/) 是分享漏洞和攻击事件的平台。

![image-20220312180116434](https://gitee.com/learnerLj/typora/raw/master/202203121801694.png)

-  [EIP-1470 提出的漏洞分类](https://swcregistry.io/)
-  [CTF 竞赛中合约安全方面的题目](https://github.com/blockthreat/blocksec-ctfs)

![image-20220312173352870](https://gitee.com/learnerLj/typora/raw/master/202203121733274.png)

-  [找合约的漏洞的挑战](https://github.com/blockthreat/blocksec-ctfs)

![image-20220312173315638](https://gitee.com/learnerLj/typora/raw/master/202203121733995.png)

- [合约安全游戏](https://capturetheether.com/)

![image-20220312173505157](https://gitee.com/learnerLj/typora/raw/master/202203121735388.png)



### 各语言的区块链开发

以下资源包来自[以太坊官网](https://ethereum.org/zh/developers/docs/programming-languages/)，主要是各种语言参与区块链开发的指导，虽然许多没有翻译，但是仍然很有参考意义，因此搬运过来了：

根据您的语言选择项目资源和虚拟社区：

- [面向 Java 开发者的以太坊资源](https://ethereum.org/zh/developers/docs/programming-languages/java/)
- [面向 Python 开发者的以太坊资源](https://ethereum.org/zh/developers/docs/programming-languages/python/)
- [面向 JavaScript 开发者的以太坊资源](https://ethereum.org/zh/developers/docs/programming-languages/javascript/)
- [面向 Go 开发者的以太坊资源](https://ethereum.org/zh/developers/docs/programming-languages/golang/)
- [面向 Rust 开发者的以太坊资源](https://ethereum.org/zh/developers/docs/programming-languages/rust/)
- [面向 .NET 开发者的以太坊资源](https://ethereum.org/zh/developers/docs/programming-languages/dot-net/)
- [面向 Delphi 开发者的以太坊资源](https://ethereum.org/zh/developers/docs/programming-languages/delphi/)
- [面向 Dart 开发者的以太坊资源](https://ethereum.org/zh/developers/docs/programming-languages/dart/)

### 底层源码参考

- [简书博客](https://www.jianshu.com/u/572268941378)，最初发表的文章主要是以太坊源码分析，对以太坊的函数做了说明。
- [最近的源码分析文章](https://github.com/blockchainGuide/blockchainguide/tree/main/source_code_analysis/ethereum/%E4%BB%A5%E5%A4%AA%E5%9D%8A%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
- [GitHub - ZtesoftCS/go-ethereum-code-analysis](https://github.com/ZtesoftCS/go-ethereum-code-analysis) 4年前的源码分析汇总
- [go-ethereum 源码笔记（概览）](https://knarfeh.com/2018/03/10/go-ethereum 源码笔记（概览）/) 四年前的博客
- [Introduction · Ethereum Development with Go](https://goethereumbook.org/zh/) 跟着用 Go 写简单区块链
- [Geth Documentation | Go Ethereum](https://geth.ethereum.org/docs/) geth 使用说明，调试程序，使用自带工具的参考。
- [操作码详解和模拟](https://www.evm.codes/)。
- [以太坊技术与实现](https://learnblockchain.cn/books/geth/)，作者作了整体性的说明，适合作为大致参考。
- [笔者的源码分析和理论基础、调试实操](https://github.com/learnerLj/geth-analyze)。



## 参考资料

- [smart-contract-development-best-practices](https://yos.io/2019/11/10/smart-contract-development-best-practices/)
- https://github.com/blockchainGuide/blockchainguide
- https://ethereum.org/zh/

