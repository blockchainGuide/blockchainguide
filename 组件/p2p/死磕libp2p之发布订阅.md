# 什么是发布订阅

发布/订阅是一个系统，同行们聚集在他们感兴趣的主题周围。据说对某个主题感兴趣的同行们订阅了该主题。

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/subscribed_peers.png)

同行可以向主题发送消息。每条消息都将发送给订阅该主题的所有对等方：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/message_delivered_to_all.png)

- 聊天室。每个房间都是一个酒吧/子主题，客户发布聊天消息，房间里的所有其他客户都会收到这些消息。

- 文件共享。每个发布/子主题代表一个可以下载的文件。上传者和下载者在发布/子主题中公布他们拥有的文件片段，并协调发布/子系统之外的下载。

## 设计目标

在对等发布/子系统中，所有对等体都参与在整个网络中传递消息。对等发布/订阅系统有几种不同的设计，它们提供了不同的权衡。理想的属性包括：

可靠性：所有消息都将传递给订阅该主题的所有对等方。

速度：信息传递迅速。

效率：网络中没有过多的消息副本。

弹性：对等方可以在不中断网络的情况下加入和离开网络。没有故障的中心点。

规模：主题可以有**大量的订阅者，并处理大量的消息**。

简单性：该系统易于理解和实现。每个对等体只需要记住一小部分状态。

ibp2p目前使用的是一种名为“八卦”的设计。它之所以命名，是因为同行们互相八卦他们看到了哪些消息，并利用这些信息来维护消息传递网络。

## 发现

在对等方可以订阅某个主题之前，它必须找到其他对等方并与其建立网络连接。pub/sub系统本身没有任何发现对等点的方法。相反，它依靠应用程序来代表自己寻找新的对等点，这一过程称为环境对等点发现。

发现对等点的潜在方法包括：

- 分布式哈希表

- 本地网络广播

- 与现有对等方交换对等方列表

- 集中式跟踪器或集合点

- 引导对等程序列表

例如，在Bittorrent应用程序中，上述大多数方法都将在下载文件的过程中使用。通过重复使用Bittorrent应用程序有关其常规业务时发现的同行，该应用程序也可以建立一个强大的酒吧/子网络。

询问发现的同行是否支持酒吧/子协议，如果是的，则将其添加到酒吧/子网络中。

## peer 类型

在Gossipsub中，对等方通过全信息的同伴或仅限元数据的同伴相互连接。整体网络结构由这两个网络组成：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/types_of_peering.png)

## 完整的消息

全消息节点对被用于在整个网络中传输消息的完整内容。这个网络是稀疏连接的，每个节点只连接了几个其他节点。（在gossipsub规范中，这个稀疏连接的网络被称为网格，其中的节点称为网格成员。）

限制完整消息节点对的数量很有用，因为它可以控制网络的流量量。每个节点只向少数几个节点转发消息，而不是所有节点。每个节点都有一个它想要连接的目标节点数。在这个例子中，每个节点理想情况下想要连接3个其他节点，但也可以接受2-4个连接：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/full_message_network.png)

节点度数（也称为网络度数或D）控制了网络速度、可靠性、弹性和效率之间的权衡。更高的节点度数有助于更快地传递消息，有更好的机会到达所有订阅者，并且更少的机会被其他节点离开时干扰网络。然而，高节点度数也会导致每条消息的额外冗余副本在整个网络中发送，增加了参与网络所需的带宽。

在libp2p的默认实现中，理想的网络节点度数为6，接受4-12个节点度数都可以。

## 仅元数据

除了全消息节点对的稀疏连接网络，还有一个密集连接的仅元数据节点对网络。这个网络由节点之间所有不是全消息节点对的网络连接组成。

仅元数据网络分享有关可用消息的八卦，并执行帮助维护全消息节点对网络的功能。

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/metadata_only_network.png)

## 嫁接和修剪

节点对是双向的，这意味着对于任意两个连接的节点，两个节点都认为它们之间的连接是全消息或是两个节点都认为它们之间的连接是仅元数据。

任何一个节点都可以通过通知另一个节点来更改连接类型。嫁接是将仅元数据连接转换为全消息的过程。修剪是相反的过程；将全消息节点对转换为仅元数据：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/graft_prune.png)

当一个节点有太少的全消息节点对时，它会随机将一些仅元数据节点对转换为全消息节点对：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/maintain_graft.png)

相反地，当一个节点有太多的全消息节点对时，它会随机地将其中一些节点对修剪为仅元数据节点对：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/maintain_prune.png)

在libp2p的实现中，每个节点每1秒执行一系列检查。这些检查称为心跳。在此期间进行嫁接和修剪操作。

## 订阅和取消订阅

节点会跟踪其直接连接的节点订阅的主题。利用这些信息，每个节点可以建立其周围主题和订阅每个主题的节点的图像:

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/subscriptions_local_view.png)

跟踪订阅是通过发送subscribe（订阅）和unsubscribe（取消订阅）消息来实现的。当两个节点之间建立新的连接时，它们首先彼此发送订阅主题的列表：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/subscription_list_first_connect.png)

然后随着时间的推移，每当一个节点订阅或取消订阅某个主题，它会向每个节点发送subscribe或unsubscribe消息。这些消息会发送给所有连接的节点，无论接收节点是否订阅了相关主题：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/subscribe_graft.png)

当一个节点取消订阅某个主题时，它会同时通知其全消息节点对其连接已被修剪，同时发送其unsubscribe消息：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/unsubscribe_prune.png)

## 发送消息

当一个节点想要发布一条消息时，它将向其连接的所有全消息节点发送一个副本：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/full_message_send.png)

同样，当一个节点从另一个节点接收到一条新消息时，它会存储消息并将副本转发给其连接的所有其他全消息节点：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/full_message_forward.png)

在Gossipsub规范中，节点也被称为路由器，因为它们通过网络路由消息的功能。

节点会记住最近收到的消息清单。这使得节点只有在第一次看到消息时才会对其采取行动，并忽略已经收到的消息的重发。

节点可能还会选择验证每个接收到的消息的内容。什么是有效和无效的，取决于应用程序。例如，聊天应用程序可能强制要求所有消息都必须短于100个字符。如果应用程序告诉libp2p某个消息无效，那么该消息将被丢弃，并且不会在整个网络中继续被复制。



## gossip

节点会在网络中广播它们最近收到的消息。每秒钟，每个节点会随机选择6个仅有元数据的节点，并向它们发送自己最近收到的消息列表。

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/gossip_deliver.png)

广播消息的目的是让节点有机会注意到它们在全消息网络中是否错过了某条消息。如果一个节点发现自己重复错过消息，则可以与已经拥有这些消息的节点建立新的全消息连接。

以下是具体消息如何在仅有元数据连接中请求的示例：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/request_gossiped_message.png)

在Gossipsub规范中，广播最近收到的消息被称为IHAVE消息，请求特定消息的消息被称为IWANT消息。


## Fan_out

节点可以发布到它们未订阅的主题。有一些特殊的规则可以确保这些消息能够可靠地传递。

当一个节点第一次想要向一个它未订阅的主题发布一条消息时，它会随机选择6个已订阅该主题的节点（下面显示了其中的3个），并将它们作为该主题的扇出节点进行记忆：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/fanout_initial_pick.png)

与其他类型的连接不同，扇出连接是单向的；它们总是从主题外的节点指向订阅主题的节点。订阅该主题的节点不会被告知它们已被选中，并且仍将该连接视为任何其他仅有元数据的连接。

每当发送方想要发送一条消息时，它会将消息发送给它的扇出节点，然后在主题内分发该消息。

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/fanout_message_send.png)

如果发送方要发送一条消息，但注意到上次以来它的一些扇出节点已经离开了，它会随机选择其他扇出节点，将数量补充至6个。

当节点订阅一个主题时，如果它已经有了一些扇出节点，它会优先选择它们作为全消息节点：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/fanout_grafting_preference.png)

如果在某个主题上连续2分钟没有发送任何消息，所有该主题的扇出节点都将被遗忘：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/fanout_forget.png)



## Network packets

节点在网络上实际发送给对方的数据包是本指南中涉及到的所有不同消息类型的组合（应用消息、有/想拥有、订阅/取消订阅、接受/拒绝）集合。这种结构允许将多种不同的请求批量发送到单个的网络数据包中。

以下是网络数据包结构的图示表示：

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/network_packet_structure.png)

请参考规范以了解用于编码网络数据包的确切 Protocol Buffers 模式。

## 状态

以下是每个节点在参与发布/订阅网络时必须记住的状态摘要：

订阅：已订阅的主题列表。
扇出主题：这些是最近发布过但未订阅的主题。对于每个主题，记住上次发送到该主题的时间。
当前连接的对等方列表：对于每个已连接的对等方，状态包括它们订阅的所有主题以及每个主题的连接方式是完整内容、仅元数据还是扇出。
最近查看的消息：这是一个最近查看的消息缓存。它用于检测和忽略重新发送的消息。对于每条消息，状态包括发送者和序列号，这足以唯一标识任何消息。对于非常近期的消息，仍然保留完整消息内容，因此可以将其发送给请求该消息的任何对等方。

![img](https://docs.libp2p.io/concepts/assets/publish-subscribe/state.png)