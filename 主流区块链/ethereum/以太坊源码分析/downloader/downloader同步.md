## 概览

syncer()

pm.peers.Len() < minDesiredPeerCount Peers里面至少5个节点

td是什么

## 同步模式

 ### full  sync

full 模式会在数据库中保存所有区块数据，同步时从远程节点同步 header 和 body 数据，而state 和 receipt 数据则是在本地计算出来的。

在 full 模式下，downloader 会同步区块的 header 和 body 数据组成一个区块，然后通过 blockchain 模块的 `BlockChain.InsertChain` 向数据库中插入区块。在 `BlockChain.InsertChain` 中，会逐个计算和验证每个块的 `state` 和 `recepit` 等数据，如果一切正常就将区块数据以及自己计算得到的 `state`、`recepit` 数据一起写入到数据库中。

### fast sync 

 `fast` 模式下，`recepit` 不再由本地计算，而是和区块数据一样，直接由 `downloader` 从其它节点中同步；`state` 数据并不会全部计算和下载，而是选一个较新的区块（称之为 `pivot`）的 `state` 进行下载，以这个区块为分界，之前的区块是没有 `state` 数据的，之后的区块会像 `full` 模式下一样在本地计算 `state`。因此在 `fast` 模式下，同步的数据除了 `header` 和 body，还有 `receipt`，以及 `pivot` 区块的 `state`。

因此 `fast` 模式忽略了大部分 `state` 数据，并且使用网络直接同步 `receipt` 数据的方式替换了 full 模式下的本地计算，所以比较快。

### light sync 

light 模式也叫做轻模式，它只对区块头进行同步，而不同步其它的数据。

SyncMode:

- FullSync:从完整区块同步整个区块链历史
- FastSync:快速下载标题，仅在链头处完全同步
- LightSync:仅下载标题，然后终止

## 区块下载流程



## synchronise

①：确保对方的TD高于我们自己的TD

```go
currentBlock := pm.blockchain.CurrentBlock()
	td := pm.blockchain.GetTd(currentBlock.Hash(), currentBlock.NumberU64())
	pHead, pTd := peer.Head()
	if pTd.Cmp(td) <= 0 {
		return
	}
```

②：开启downloader的同步

```go
pm.downloader.Synchronise(peer.id, pHead, pTd, mode)
```

进入函数：主要做了以下几件事：

1. d.synchronise(id, head, td, mode) ：同步过程
2. 错误日志输出， 并删除此peer。

```go

```

-------





## findAncestor

同步首要的是**确定同步区块的区间**：顶部为远程节点的最高区块，底部为两个节点都拥有的相同区块的最高高度（祖先区块）。`findAncestor`就是用来找祖先区块。函数分析如下：

①：确定本地高度和远程节点的最高高度

```go
var (
		floor        = int64(-1) // 底部
		localHeight  uint64  // 本地最高高度
		remoteHeight = remoteHeader.Number.Uint64() // 远程节点最高高度
	)
switch d.mode {
	case FullSync:
		localHeight = d.blockchain.CurrentBlock().NumberU64()
	case FastSync:
		localHeight = d.blockchain.CurrentFastBlock().NumberU64()
	default:
		localHeight = d.lightchain.CurrentHeader().Number.Uint64()
	}
```

②：计算同步的高度区间和间隔

```go
from, count, skip, max := calculateRequestSpan(remoteHeight, localHeight) 
```

- `from`:：表示从哪个高度开始获取区块
- `count`：表示从远程节点获取多少个区块
- `skip`：表示间隔，比如`skip` 为 2 ，获取第一个高度为 5，则第二个就是 8
- `max`：表示最大高度

③：发送获取`header`的请求

```go
go p.peer.RequestHeadersByNumber(uint64(from), count, skip, false)
```

④：处理上面请求接收到的`header`  :`case packet := <-d.headerCh`

1. 丢弃掉不是来自我们请求节的内容
2. 确保返回的`header`数量不为空
3. 验证返回的`headers`的高度是我们所请求的
4. 检查是否找到共同祖先

```go
//----①
if packet.PeerId() != p.id {
				log.Debug("Received headers from incorrect peer", "peer", packet.PeerId())
				break
			}
//-----②
headers := packet.(*headerPack).headers
			if len(headers) == 0 {
				p.log.Warn("Empty head header set")
        return 0
      }
//-----③
for i, header := range headers {
				expectNumber := from + int64(i)*int64(skip+1)
				if number := header.Number.Int64(); number != expectNumber { // 验证这些返回的header是否是我们上面请求的headers
					p.log.Warn("Head headers broke chain ordering", "index", i, "requested", expectNumber, "received", number)
					return 0, errInvalidChain
				}
			}
//-----④
// Check if a common ancestor was found 检查是否找到共同祖先
			finished = true
			//注意这里是从headers最后一个元素开始查找，也就是高度最高的区块。
			for i := len(headers) - 1; i >= 0; i-- {
				// Skip any headers that underflow/overflow our requested set
				// 跳过不在我们请求的高度区间内的区块
				if headers[i].Number.Int64() < from || headers[i].Number.Uint64() > max {
					continue
				}
				// Otherwise check if we already know the header or not
				// //检查我们本地是否已经有某个区块了，如果有就算是找到了共同祖先，
				//并将共同祖先的哈希和高度设置在number和hash变量中。
				h := headers[i].Hash()
				n := headers[i].Number.Uint64()

        
        
```

⑤：如果通过固定间隔法找到了共同祖先则返回祖先，会对其高度与 `floor` 变量进行验证, `floor` 变量代表的是共同祖先的高度的最小值,如果找到共同祖先的高度比这个值还小，就认为是两个节点之间分叉太大了，不再允许进行同步。如果一切正常，就返回找到的共同祖先的高度 `number` 变量。

```GO
if hash != (common.Hash{}) {
        if int64(number) <= floor {
            return 0, errInvalidAncestor
        }
        return number, nil
    }
```

⑥：如果固定间隔法没有找到祖先则通过二分法来查找祖先，这部分可以思想跟二分法算法类似，有兴趣的可以细看。

-----

## fetchHeaders

`fetchHeaders`的大致思想：

 同步header的数据会被填充到`skeleton`，每次从远程节点获取区块数据最大为`MaxHeaderFetch`（192），所以要获取的区块数据如果大于192 ，会被分成组，每组`MaxHeaderFetch`，剩余的不足192个的不会填充进`skeleton`，具体步骤如下图所示：

![image-20201209165112013](https://tva1.sinaimg.cn/large/0081Kckwgy1glhpclqzngj314k0megpn.jpg)

此种方式可以**避免从同一节点下载过多错误数据**，如果我们连接到了一个恶意节点，它可以创造一个链条很长且TD值也非常高的区块链数据。如果我们的区块从 0 开始全部从它那同步，也就下载了一些根本不被别人承认的数据。如果我只从它那同步 `MaxHeaderFetch` 个区块，然后发现这些区块无法正确填充我之前的 `skeleton`（可能是 `skeleton` 的数据错了，或者用来填充 `skeleton` 的数据错了），就会丢掉这些数据。











## 参考

> https://mindcarver.cn
>
> https://github.com/ethereum/go-ethereum/pull/1889

