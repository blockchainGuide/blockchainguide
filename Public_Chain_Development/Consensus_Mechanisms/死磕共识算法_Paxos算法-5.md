> 死磕共识算法|Paxos算法
>
> 配合以下代码进行阅读：https://github.com/blockchainGuide/
>
> 写文不易，给个小关注，有什么问题可以指出，便于大家交流学习。

![b9a24d7b935c61ba4555b1ddc8159413](https://tva1.sinaimg.cn/large/008eGmZEgy1gnuvfqrojoj312w0m8119.jpg)



# Paxos是什么

> Paxos算法是基于**消息传递**且具有**高度容错特性**的**一致性算法**，是目前公认的解决**分布式一致性**问题**最有效**的算法之一。

`Paxos`由`Lamport`于1998年在《The Part-Time Parliament》论文中首次公开，最初的描述使用希腊的一个小岛`Paxos`作为比喻，描述了`Paxos`小岛中通过决议的流程，并以此命名这个算法，但是这个描述理解起来比较有挑战性。后来在2001年，`Lamport`觉得同行不能理解他的幽默感，于是重新发表了朴实的算法描述版本《Paxos Made Simple》。

自`Paxos`问世以来就持续垄断了分布式一致性算法，`Paxos`这个名词几乎等同于分布式一致性。`Google`的很多大型分布式系统都采用了`Paxos`算法来解决分布式一致性问题。

# Paxos相关概念

Paxos算法运行在允许宕机故障的异步系统中，不要求可靠的消息传递，可容忍消息丢失、延迟、乱序以及重复。它利用大多数 (Majority) 机制保证了2F+1的容错能力，即**2F+1**个节点的系统最多允许**F**个节点同时出现故障。

一个或多个提议进程 (Proposer) 可以发起提案 (Proposal)，Paxos算法使所有提案中的某一个提案，在所有进程中达成一致。系统中的多数派同时认可该提案，即达成了一致。最多只针对一个确定的提案达成一致。

Paxos将系统中的角色分为提议者 (Proposer)，决策者 (Acceptor)，和最终决策学习者 (Learner):

- **Proposer**: 提出提案 (Proposal)。Proposal信息包括提案编号 (Proposal ID) 和提议的值 (Value)。
- **Acceptor**：参与决策，回应Proposers的提案。收到Proposal后可以接受提案，若Proposal获得多数Acceptors的接受，则称该Proposal被批准。
- **Learner**：不参与决策，从Proposers/Acceptors学习最新达成一致的提案（Value）。

在多副本状态机中，每个副本同时具有Proposer、Acceptor、Learner三种角色。

![image-20210221103652296](https://tva1.sinaimg.cn/large/008eGmZEgy1gnuydua826j31gg0i6q5j.jpg)

# paxos算法流程

Paxos算法通过一个决议分为两个阶段（Learn阶段之前决议已经形成）：

1. 第一阶段：Prepare阶段。Proposer向Acceptors发出Prepare请求，Acceptors针对收到的Prepare请求进行Promise承诺。
2. 第二阶段：Accept阶段。Proposer收到多数Acceptors承诺的Promise后，向Acceptors发出Propose请求，Acceptors针对收到的Propose请求进行Accept处理。
3. 第三阶段：Learn阶段。Proposer在收到多数Acceptors的Accept之后，标志着本次Accept成功，决议形成，将形成的决议发送给所有Learners。

![image-20210221104706981](https://tva1.sinaimg.cn/large/008eGmZEgy1gnuyoi6keij318e0isady.jpg)

Paxos算法流程中的每条消息描述如下：

- **Prepare**: Proposer生成全局唯一且递增的Proposal ID (可使用时间戳加Server ID)，向所有Acceptors发送Prepare请求，这里无需携带提案内容，只携带Proposal ID即可。
- **Promise**: Acceptors收到Prepare请求后，做出“两个承诺，一个应答”。

**两个承诺：**

1. 不再接受Proposal ID小于等于（注意：这里是<= ）当前请求的Prepare请求。

2. 不再接受Proposal ID小于（注意：这里是< ）当前请求的Propose请求。

**一个应答：**

不违背以前作出的承诺下，回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值。

- **Propose**: Proposer 收到多数Acceptors的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。如果所有应答的提案Value均为空值，则可以自己随意决定提案Value。然后携带当前Proposal ID，向所有Acceptors发送Propose请求。
- **Accept**: Acceptor收到Propose请求后，在不违背自己之前作出的承诺下，接受并持久化当前Proposal ID和提案Value。
- **Learn**: Proposer收到多数Acceptors的Accept后，决议形成，将形成的决议发送给所有Learners。

伪代码流程如下：

1. 获取一个Proposal ID n，为了**保证Proposal ID唯一**，可采用时间戳+Server ID生成；
2. Proposer向所有Acceptors广播Prepare(n)请求；
3. Acceptor比较n和minProposal，如果n>minProposal，minProposal=n，并且将 acceptedProposal 和 acceptedValue 返回；
4. Proposer接收到过半数回复后，如果发现有acceptedValue返回，将所有回复中acceptedProposal最大的acceptedValue作为本次提案的value，否则可以任意决定本次提案的value；
5. 到这里可以进入第二阶段，广播Accept (n,value) 到所有节点；
6. Acceptor比较n和minProposal，如果n>=minProposal，则acceptedProposal=minProposal=n，acceptedValue=value，本地持久化后，返回；否则，返回minProposal。
7. 提议者接收到过半数请求后，如果发现有返回值result >n，表示有更新的提议，跳转到1；否则value达成一致。



# 案例分析

**案例①：**

图中P代表Prepare阶段，A代表Accept阶段。3.1代表Proposal ID为3.1，其中3为时间戳，1为Server ID。X和Y代表提议Value。

实例1中P 3.1达成多数派，其Value(X)被Accept，然后P 4.5学习到Value(X)，并Accept。

![image-20210221124740017](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv264l27aj31ce0l4jy9.jpg)

**案例②：**

实例2中P 3.1没有被多数派Accept（只有S3 Accept），但是被P 4.5学习到，P 4.5将自己的Value由Y替换为X，Accept（X）。

![image-20210221124945254](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv283ux5xj31bi0gydkn.jpg)

**案例③：**

实例3中P 3.1没有被多数派Accept（只有S1 Accept），同时也没有被P 4.5学习到。由于P 4.5 Propose的所有应答，均未返回Value，则P 4.5可以Accept自己的Value (Y)。后续P 3.1的Accept (X) 会失败，已经Accept的S1，会被覆盖。

![image-20210221125030450](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv28vsafmj318q0fsjw8.jpg)

Paxos算法可能形成活锁而永远不会结束，如下图实例所示：

![image-20210221125102677](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv29ol30oj319y0fyafm.jpg)

回顾两个承诺之一，Acceptor不再应答Proposal ID小于等于当前请求的Prepare请求。意味着需要应答Proposal ID大于当前请求的Prepare请求。

两个Proposers交替Prepare成功，而Accept失败，形成活锁（Livelock）。

# Multi-Paxos算法

原始的Paxos算法（Basic Paxos）**只能对一个值形成决议**，决议的形成至少需要两次网络来回，在高并发情况下可能需要更多的网络来回，极端情况下甚至可能形成活锁。如果想连续确定多个值，Basic Paxos搞不定了。因此Basic Paxos几乎只是用来做理论研究，并不直接应用在实际工程中。

**实际应用中几乎都需要连续确定多个值，而且希望能有更高的效率。Multi-Paxos正是为解决此问题而提出**。Multi-Paxos基于Basic Paxos做了两点改进：

1. 针对每一个要确定的值，运行一次Paxos算法实例（Instance），形成决议。每一个Paxos实例使用唯一的Instance ID标识。
2. 在所有Proposers中选举一个Leader，由Leader唯一地提交Proposal给Acceptors进行表决。这样**没有Proposer竞争**，解决了活锁问题。在系统中仅有一个Leader进行Value提交的情况下，Prepare阶段就可以跳过，从而将两阶段变为一阶段，提高效率。

![image-20210221125526065](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv2e0hvd4j31bs0i00wn.jpg)

Multi-Paxos首先需要选举Leader，Leader的确定也是一次决议的形成，所以可执行一次Basic Paxos实例来选举出一个Leader。选出Leader之后只能由Leader提交Proposal，在Leader宕机之后服务临时不可用，需要重新选举Leader继续服务。在系统中仅有一个Leader进行Proposal提交的情况下，Prepare阶段可以跳过。

Multi-Paxos通过改变Prepare阶段的作用范围至后面Leader提交的所有实例，从而使得Leader的连续提交只需要执行一次Prepare阶段，后续只需要执行Accept阶段，将两阶段变为一阶段，提高了效率。为了区分连续提交的多个实例，每个实例使用一个Instance ID标识，Instance ID由Leader本地递增生成即可。

Multi-Paxos允许有多个自认为是Leader的节点并发提交Proposal而不影响其安全性，这样的场景即退化为Basic Paxos。

# Paxos推导过程

## 只有一个Acceptor

假设只有一个Acceptor（可以有多个Proposer），只要Acceptor接受它收到的第一个提案，则该提案被选定，该提案里的value就是被选定的value。这样就保证只有一个value会被选定。

但是，如果这个唯一的Acceptor宕机了，那么整个系统就**无法工作**了！

![image-20210221130224968](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv2lac3xej313w0gowge.jpg)

## 多个Acceptor

多个`Acceptor`需要保证在多个`Proposer`和多个`Acceptor`的情况下选定一个`value`。

![image-20210221130706677](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv2q6ctcqj31b80i8415.jpg)

如果我们希望即使只有一个Proposer提出了一个value，该value也最终被选定。

那么，就得到下面的约束：

> P1：一个Acceptor必须接受它收到的第一个提案。

但是，这又会引出另一个问题：如果每个Proposer分别提出不同的value，发给不同的Acceptor。根据P1，Acceptor分别接受自己收到的value，就导致不同的value被选定。出现了不一致。如下图：

![image-20210221131721430](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv30tvh7ij31dy0pcju7.jpg)

刚刚是因为『一个提案只要被一个Acceptor接受，则该提案的value就被选定了』才导致了出现上面不一致的问题。因此，我们需要加一个规定：

> 规定：一个提案被选定需要被**半数以上**的Acceptor接受

这个规定又暗示了：『一个Acceptor必须能够接受不止一个提案！』不然可能导致最终没有value被选定。比如上图的情况。v1、v2、v3都没有被选定，因为它们都只被一个Acceptor的接受。

最开始讲的『**提案=value**』已经不能满足需求了，于是重新设计提案，给每个提案加上一个提案编号，表示提案被提出的顺序。令『**提案=提案编号+value**』。

虽然允许多个提案被选定，但必须保证所有被选定的提案都具有相同的value值。否则又会出现不一致。

于是有了下面的约束：

> P2：如果某个value为v的提案被选定了，那么每个编号更高的被选定提案的value必须也是v。

一个提案只有被Acceptor接受才可能被选定，因此我们可以把P2约束改写成对Acceptor接受的提案的约束P2a。

> P2a：如果某个value为v的提案被选定了，那么每个编号更高的被Acceptor接受的提案的value必须也是v。

只要满足了P2a，就能满足P2。

但是，考虑如下的情况：假设总的有5个Acceptor。Proposer2提出[M1,V1]的提案，Acceptor25（半数以上）均接受了该提案，于是对于Acceptor25和Proposer2来讲，它们都认为V1被选定。Acceptor1刚刚从宕机状态恢复过来（之前Acceptor1没有收到过任何提案），此时Proposer1向Acceptor1发送了[M2,V2]的提案（V2≠V1且M2>M1），对于Acceptor1来讲，这是它收到的第一个提案。根据P1（一个Acceptor必须接受它收到的第一个提案。）,Acceptor1必须接受该提案！同时Acceptor1认为V2被选定。这就出现了两个问题：

1. Acceptor1认为V2被选定，Acceptor2~5和Proposer2认为V1被选定。出现了不一致。
2. V1被选定了，但是编号更高的被Acceptor1接受的提案[M2,V2]的value为V2，且V2≠V1。这就跟P2a（如果某个value为v的提案被选定了，那么每个编号更高的被Acceptor接受的提案的value必须也是v）矛盾了。

![image-20210221133102803](https://tva1.sinaimg.cn/large/008eGmZEgy1gnv3f3co1cj31gs0pk77y.jpg)

P2a是对Acceptor接受的提案约束，但其实提案是Proposer提出来的，所有我们可以对Proposer提出的提案进行约束。得到P2b：

> P2b：如果某个value为v的提案被选定了，那么之后任何Proposer提出的编号更高的提案的value必须也是v。

由P2b可以推出P2a进而推出P2。

那么，如何确保在某个value为v的提案被选定后，Proposer提出的编号更高的提案的value都是v呢？

只要满足P2c即可：

> P2c：对于任意的N和V，如果提案[N, V]被提出，那么存在一个半数以上的Acceptor组成的集合S，满足以下两个条件中的任意一个：

- S中每个Acceptor都没有接受过编号小于N的提案。
- S中Acceptor接受过的最大编号的提案的value为V。

## Proposer生成提案

为了满足P2b，这里有个比较重要的思想：Proposer生成提案之前，应该先去**『学习』**已经被选定或者可能被选定的value，然后以该value作为自己提出的提案的value。如果没有value被选定，Proposer才可以自己决定value的值。这样才能达成一致。这个学习的阶段是通过一个**『Prepare请求』**实现的。

于是我们得到了如下的**提案生成算法**：

1. Proposer选择一个**新的提案编号N**，然后向**某个Acceptor集合**（半数以上）发送请求，要求该集合中的每个Acceptor做出如下响应（response）。
   (a) 向Proposer承诺保证**不再接受**任何编号**小于N的提案**。
   (b) 如果Acceptor已经接受过提案，那么就向Proposer响应**已经接受过**的编号小于N的**最大编号的提案**。

   我们将该请求称为**编号为N**的**Prepare请求**。

2. 如果Proposer收到了**半数以上**的Acceptor的**响应**，那么它就可以生成编号为N，Value为V的**提案[N,V]**。这里的V是所有的响应中**编号最大的提案的Value**。如果所有的响应中**都没有提案**，那 么此时V就可以由Proposer**自己选择**。
   生成提案后，Proposer将该**提案**发送给**半数以上**的Acceptor集合，并期望这些Acceptor能接受该提案。我们称该请求为**Accept请求**。（注意：此时接受Accept请求的Acceptor集合**不一定**是之前响应Prepare请求的Acceptor集合）

## 为什么需要 Propose 阶段

因为对 paxos 来说，是假定一个集群中会有多个paxos instance（也就是多个提案）同时存在竞争的（并发冲突）。那么 propose 阶段就是选择出需要进行投票的paxos instance。如果能够保证只有一个paxos instance，那么就无需 propose 阶段了，直接进行accept即可。所以对于multi-paxos中，存在一个leader，可以控制每个时刻只有一个paxos instance在集群中，所以不需要propose阶段，只需要执行accept阶段即可。

这里就相当于一个add 1 的paxos instance，一个 delete key 的paxos instance。只有当整个集群指定的 paxos instance 的顺序是相同的，也就是，也就是每个节点都是先add 1，然后在 delete key，或者先delete key，再add 1，最后的数据才会一致。它本质上解决的就是有多个议案的情况下， 达成一个一致的议案，例如，一群人决定聚餐，有想吃鱼的，想吃火锅的，这样多个决议进行 paxos 提案投票，就会得到一个一致的聚餐结果。如果没有多个决议，只有一个决议，那就不会冲突，直接accept投票即可。

**Paxos Propose 的意义**

1. **Block old proposals**
2. **Find out about (possibly) accepted values**

## Acceptor接受提案

Acceptor**可以忽略任何请求**（包括Prepare请求和Accept请求）而不用担心破坏算法的**安全性**。因此，我们这里要讨论的是什么时候Acceptor可以响应一个请求。

我们对Acceptor接受提案给出如下约束：

> P1a：一个Acceptor只要尚**未响应过**任何**编号大于N**的**Prepare请求**，那么他就可以**接受**这个**编号为N的提案**。

如果Acceptor收到一个编号为N的Prepare请求，在此之前它已经响应过编号大于N的Prepare请求。根据P1a，该Acceptor不可能接受编号为N的提案。因此，该Acceptor可以忽略编号为N的Prepare请求。当然，也可以回复一个error，让Proposer尽早知道自己的提案不会被接受。

因此，一个Acceptor**只需记住**：1. 已接受的编号最大的提案 2. 已响应的请求的最大编号。

------

# 参考

>  https://github.com/blockchainGuide
>
>  公号：区块链技术栈
>
>  [http://zh.wikipedia.org/zh-cn/Paxos%E7%AE%97%E6%B3%95](http://zh.wikipedia.org/zh-cn/Paxos算法)
>
>  https://www.cnblogs.com/linbingdong/p/6253479.html
>
>  https://zhuanlan.zhihu.com/p/31780743
>
>  https://www.jianshu.com/go-wild?ac=2&url=https%3A%2F%2Fblog.csdn.net%2Fsparkliang%2Farticle%2Fdetails%2F5740882
>
>  https://www.jianshu.com/go-wild?ac=2&url=http%3A%2F%2Fresearch.microsoft.com%2Fen-us%2Fum%2Fpeople%2Flamport%2Fpubs%2Fpaxos-simple.pdf
>
>  https://www.jianshu.com/go-wild?ac=2&url=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FPaxos_%28computer_science%29
>
>  https://www.jianshu.com/p/06a477a576bf