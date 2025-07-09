> 死磕共识算法|实用拜占庭容错算法(PBFT)
>
> 配合以下代码进行阅读：https://github.com/blockchainGuide/
>
> 写文不易，给个小关注，有什么问题可以指出，便于大家交流学习。

![816246ba75a183d188bdd3bf9c5eed69](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv7uqdgnmj31hc0u0wk5.jpg)

# 拜占庭容错BFT

拜占庭容错是分布式协议的一种属性，如果这种协议可以解决不可信任环境下的分布式一致性问题，那么它就是拜占庭容错。

pbft 算法的提出主要是为了解决拜占庭将军问题。网上关于 pbft 的算法介绍基本上是基于 liskov 在 1999 年发表的论文《 Practical Byzantine Fault Tolerance 》来进行解释的。



# raft和pbft的最大容错节点数

对于raft算法，raft算法只支持容错故障节点，不支持容错作恶节点。假设集群总节点数为n，故障节点为 f ，集群里正常节点只需要比 f 个节点再多一个节点，即 f+1 个节点，正确节点的数量就会比故障节点数量多，那么集群就能达成共识。因此 raft 算法支持的最大容错节点数量是（n-1）/2。

对于 pbft 算法，因为 pbft 算法的除了需要支持容错故障节点之外，还需要支持容错作恶节点。假设集群节点数为 N，有问题的节点为 f。有问题的节点中，可以既是故障节点，也可以是作恶节点，或者只是故障节点或者只是作恶节点。那么会产生以下两种极端情况：

1. 第一种情况，f 个有问题节点既是故障节点，又是作恶节点，那么根据小数服从多数的原则，集群里正常节点只需要比f个节点再多一个节点，即 f+1 个节点，确节点的数量就会比故障节点数量多，那么集群就能达成共识。也就是说这种情况支持的最大容错节点数量是 （n-1）/2。
2. 第二种情况，故障节点和作恶节点都是不同的节点。那么就会有 f 个问题节点和 f 个故障节点，当发现节点是问题节点后，会被集群排除在外，剩下 f 个故障节点，那么根据小数服从多数的原则，集群里正常节点只需要比f个节点再多一个节点，即 f+1 个节点，确节点的数量就会比故障节点数量多，那么集群就能达成共识。所以，所有类型的节点数量加起来就是 f+1 个正确节点，f个故障节点和f个问题节点，即 3f+1=n。

结合上述两种情况，因此 pbft 算法支持的最大容错节点数量是（n-1）/3。



# PBFT算法流程

基本流程如下：

1. 客户端发送请求给主节点 
2. 主节点广播请求给其它节点，节点执行 pbft 算法的三阶段共识流程。
3. 节点处理完三阶段流程后，返回消息给客户端。
4. 客户端收到来自 f+1 个节点的相同消息后，代表共识已经正确完成。

无论是最好的情况还是最坏的情况，如果客户端收到 f+1 个节点的相同消息，那么就代表有足够多的正确节点已全部达成共识并处理完毕了。

PBFT 算法中, 存在一个主节点(primary) 和其他的备份节点 (replica), PBFT 共识机制主要包含两部分: 第一部分是分布式共识达成,在主节点正常工作时, PBFT 通过预准备 (pre-prepare)、准备 (prepare) 和承诺 (commit) 三个步骤完成共识; 第二部分是视图转换 (view-change), 当主节点出现问题不能及时处理数据请求时, 其他备份节点发起视图转换, 转换成功后新的主节点开始工作. 主节点以轮转 (round robin) 的方式交替更换.

PBFT 的分布式共识达成过程如下:

1. 请求 (propose)：客户端 (client) 上传请求消息 *m* 至网络中的节点, 包括主节点和其他备份节点。
2. 预准备 (pre-prepare)：主节点收到客户端上传的请求消息 *m*, 赋予消息序列号 *s*, 计算得到预准备消息 (pre-prepare*, H*(*m*)*, s, v*)，其中 *H*(m) 是单向哈希函数, *v* 代表的是此时的视图 (view),视图一般用于记录主节点的更替, 主节点发生更替时, 视图随之增加 1 。消息发送者节点在发送消息前需利用自身私钥对消息实施数字签名。主节点将预准备消息发送给其他备份节点.
3. 准备 (prepare)：备份节点收到主节点的预准备消息, 验证 *H*(*m*) 的合法性。即对于视图 *v* 和序列号*s* 来说, 备份节点先前并未收到其他消息。验证通过后, 备份节点计算准备消息 (prepare*, H*(*m*)*, s, v*) 并将其在全网广播. 与此同时, 所有节点收集准备消息,如果收集到的合法准备消息数量大于等于 2*f* + 1(包含自身准备消息) 个, 则将其组成准备凭证 (prepared certificate)
4. 承诺 (commit)：如果在准备阶段中, 节点收集到足够的准备消息并生成了准备凭证, 那么节点将计算承诺消息 (commit*, s, v*) 并广播，将消息 *m* 放入到本地日志中. 与此同时节点收集网络中的承诺消息,如果收集到的合法承诺消息数量大于等于 2*f* +1(包含自身承诺消息), 那么将其组成承诺凭证 (committedcertificate), 证明消息 *m* 完成最终承诺。
5. 答复 (reply)：备份节点和主节点中任意收集到足够承诺消息并组成承诺凭证的节点, 将承诺凭证作为对消息 *m* 的答复发送给客户端, 客户端确认消息 *m* 的最终承诺.

PBFT的共识过程如下：

![image-20210223143130925](https://tva1.sinaimg.cn/large/008eGmZEgy1gnxgennnu6j31fu0g6aet.jpg)

# checkpoint机制

在 PBFT 中, 存在检查点 (checkpoint) 机制, 由于每个消息都被赋予了一定的序列号, 如消息 *m* 对应的序列号为 118, 当不少于 2*f* + 1 个节点组成消息 *m* 的承诺凭证, 完成消息承诺之后, 序列号 118 成为当前的稳定检查点 (stable checkpoint). 检查点机制被用于实现存储删减, 即当历史日志内容过多时, 节点可以选择清除稳定检查点之前的数据, 减少存储成本. 另外稳定检查点在 PBFT 的视图转换中也起到了关键作用.

# viewChange机制

当主节点挂了（超时无响应）或者从节点集体认为主节点是问题节点时，就会触发 ViewChange 事件， ViewChange 完成后，视图编号将会加 1 。下图展示 ViewChange 的三个阶段流程：

![image-20210223144812547](https://tva1.sinaimg.cn/large/008eGmZEgy1gnxgw54qrsj31a00dw7be.jpg)

viewchange 会有三个阶段，分别是 `view-change` ， `view-change-ack` 和 `new-view` 阶段。从节点认为主节点有问题时，会向其它节点发送 view-change 消息，当前存活的节点编号最小的节点将成为新的主节点。当新的主节点收到 2f 个其它节点的 view-change 消息，则证明有足够多人的节点认为主节点有问题，于是就会向其它节点广播 New-view 消息。注意：从节点不会发起 new-view 事件。对于主节点，发送 new-view 消息后会继续执行上个视图未处理完的请求，从 pre-prepare 阶段开始。其它节点验证 new-view 消息通过后，就会处理主节点发来的 pre-prepare 消息，这时执行的过程就是前面描述的 pbft 过程。

最后一张图来了解一下PBFT算法：

![PBFT-PR](/Users/carver/Downloads/PBFT-PR.svg)

------

# 参考

>  https://github.com/blockchainGuide
>
>  http://pmg.csail.mit.edu/papers/osdi99.pdf 
>
>  https://blog.csdn.net/shangsongwww/article/details/88942215
>
>  https://www.microsoft.com/en-us/research/wp-content/uploads/2017/01/thesis-mcastro.pdf
>
>  https://www.comp.nus.edu.sg/~rahul/allfiles/cs6234-16-pbft.pdf
>
>  https://zhuanlan.zhihu.com/p/35847127
>
>  https://www.jianshu.com/p/0bef4fb1662b
>
>  https://learnblockchain.cn/article/781（为什么需要三阶段消息）
>
>  https://lessisbetter.site/2020/03/22/why-pbft-needs-viewchange/（View Change的作用）