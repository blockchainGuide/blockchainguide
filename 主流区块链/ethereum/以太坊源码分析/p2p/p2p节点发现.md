

> 死磕以太坊源码分析之p2p节点发现

在阅读节点发现源码之前必须要理解kadmilia算法，[可以参考：KAD算法详解](https://github.com/blockchainGuide/blockchainguide/blob/main/%E4%B8%BB%E6%B5%81%E5%8C%BA%E5%9D%97%E9%93%BE/ethereum/%E4%BB%A5%E5%A4%AA%E5%9D%8A%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/p2p/KAD%E7%AE%97%E6%B3%95.md)。

## 节点发现概述

节点发现，使本地节点得知其他节点的信息，进而加入到p2p网络中。

以太坊的节点发现基于类似的kademlia算法，源码中有两个版本，v4和v5。v4适用于全节点，通过`discover.ListenUDP`使用，v5适用于轻节点通过`discv5.ListenUDP`使用，本文介绍的是v4版本。

节点发现功能主要涉及 **Server** **Table** **udp** 这几个数据结构，它们有独自的事件响应循环，节点发现功能便是它们互相协作完成的。其中，每个以太坊客户端启动后都会在本地运行一个**Server**，并将网络拓扑中相邻的节点视为**Node**，而**Table**是**Node**的容器，**udp**则是负责维持底层的连接。这些结构的关系如下图：

![image-20201123210628944](https://tva1.sinaimg.cn/large/0081Kckwgy1gkzetbpzowj30z00egtbh.jpg)

## p2p服务开启节点发现



在P2p的server.go 的start方法中:

```go
if err := srv.setupDiscovery(); err != nil {
		return err
	}
```

进入到`setupDiscovery`中：

```go
// Discovery V4
	var unhandled chan discover.ReadPacket
	var sconn *sharedUDPConn
	if !srv.NoDiscovery {
		...
		ntab, err := discover.ListenUDP(conn, srv.localnode, cfg)
		....
	}
```

`discover.ListenUDP`方法即开启了节点发现的功能.

首先解析出监听地址的UDP端口，根据端口返回与之相连的UDP连接，之后返回连接的本地网络地址，接着设置最后一个UDP-on-IPv4端口。到此为止节点发现的一些准备工作做好，接下下来开始UDP的监听：

```go
ntab, err := discover.ListenUDP(conn, srv.localnode, cfg)
```

然后进行UDP 的监听，下面是监听的过程：

### 监听UDP

```go
// 监听给定的socket 上的发现的包
func ListenUDP(c UDPConn, ln *enode.LocalNode, cfg Config) (*UDPv4, error) {
	return ListenV4(c, ln, cfg)
}
```

```go
func ListenV4(c UDPConn, ln *enode.LocalNode, cfg Config) (*UDPv4, error) {
	closeCtx, cancel := context.WithCancel(context.Background())
	t := &UDPv4{
		conn:            c,
		priv:            cfg.PrivateKey,
		netrestrict:     cfg.NetRestrict,
		localNode:       ln,
		db:              ln.Database(),
		gotreply:        make(chan reply),
		addReplyMatcher: make(chan *replyMatcher),
		closeCtx:        closeCtx,
		cancelCloseCtx:  cancel,
		log:             cfg.Log,
	}
	if t.log == nil {
		t.log = log.Root()
	}

	tab, err := newTable(t, ln.Database(), cfg.Bootnodes, t.log) // 
	if err != nil {
		return nil, err
	}
	t.tab = tab
	go tab.loop() //

	t.wg.Add(2)
	go t.loop() //
	go t.readLoop(cfg.Unhandled) //
	return t, nil
}
```

主要做了以下几件事：

#### 1.新建路由表

```go
tab, err := newTable(t, ln.Database(), cfg.Bootnodes, t.log) 
```

新建路由表做了以下几件事：

- 初始化table对象
- 设置bootnode（setFallbackNodes）
  - 节点第一次启动的时候，节点会与硬编码在以太坊源码中的`bootnode`进行连接，所有的节点加入几乎都先连接了它。连接上`bootnode`后，获取`bootnode`部分的邻居节点，然后进行节点发现，获取更多的活跃的邻居节点
  - nursery 是在 Table 为空并且数据库中没有存储节点时的初始连接节点（上文中的 6 个节点），通过 bootnode 可以发现新的邻居
- tab.seedRand：使用提供的种子值将生成器初始化为确定性状态
- loadSeedNodes：加载种子节点；从保留已知节点的数据库中随机的抽取30个节点，再加上引导节点列表中的节点，放置入k桶中，如果K桶没有空间，则假如到替换列表中。

#### 2.测试邻居节点连通性

首先知道UDP协议是没有连接的概念的，所以需要不断的ping 来测试对端节点是否正常，在新建路由表之后，就来到下面的循环，不断的去做上面的事。

```go
go tab.loop()
```

定时运行`doRefresh`、`doRevalidate`、`copyLiveNodes`进行刷新K桶。

以太坊的k桶设置：

```go
const (
	alpha           = 3  // Kademlia并发参数, 是系统内一个优化参数,控制每次从K桶最多取出节点个数,ethereum取值3
  
	bucketSize      = 16 // K桶大小(可容纳节点数)
  
	maxReplacements = 10 // 每桶更换列表的大小
	hashBits          = len(common.Hash{}) * 8 //每个节点ID长度,32*8=256, 32位16进制
	nBuckets          = hashBits / 15       //  K桶个数
  ）
```

首先搞清楚这三个定时器运行的时间：

```GO
refreshInterval    = 30 * time.Minute
revalidateInterval = 10 * time.Second
copyNodesInterval  = 30 * time.Second
```

##### `doRefresh`

doRefresh对随机目标执行查找以保持K桶已满。如果表为空（初始引导程序或丢弃的有故障），则插入种子节点。

主要以下几步：

1. 从数据库加载随机节点和引导节点。这应该会产生一些以前见过的节点

   ```GO
   tab.loadSeedNodes()
   ```

2. 将本地节点ID作为目标节点进行查找最近的邻居节点

   ```go
   tab.net.lookupSelf()
   ```

   ```go
   func (t *UDPv4) lookupSelf() []*enode.Node {
   	return t.newLookup(t.closeCtx, encodePubkey(&t.priv.PublicKey)).run()
   }
   ```

   ```go
   func (t *UDPv4) newLookup(ctx context.Context, targetKey encPubkey) *lookup {
   	...
   		return t.findnode(n.ID(), n.addr(), targetKey)
   	})
   	return it
   }
   ```

   向这些节点发起`findnode`操作查询离target节点最近的节点列表,将查询得到的节点进行`ping-pong`测试,将测试通过的节点落库保存

   经过这个流程后,节点的K桶就能够比较均匀地将不同网络节点更新到本地K桶中。

   ```go
   unc (t *UDPv4) findnode(toid enode.ID, toaddr *net.UDPAddr, target encPubkey) ([]*node, error) {
   	t.ensureBond(toid, toaddr)
   	nodes := make([]*node, 0, bucketSize)
   	nreceived := 0
     // 设置回应回调函数，等待类型为neighborsPacket的邻近节点包，如果类型对，就执行回调请求
   	rm := t.pending(toid, toaddr.IP, p_neighborsV4, func(r interface{}) (matched bool, requestDone bool) {
   		reply := r.(*neighborsV4)
   		for _, rn := range reply.Nodes {
   			nreceived++
         // 得到一个简单的node结构
   			n, err := t.nodeFromRPC(toaddr, rn)
   			if err != nil {
   				t.log.Trace("Invalid neighbor node received", "ip", rn.IP, "addr", toaddr, "err", err)
   				continue
   			}
   			nodes = append(nodes, n)
   		}
   		return true, nreceived >= bucketSize
   	})
     //上面了一个管道事件，下面开始发送真正的findnode报文，然后进行等待了
   	t.send(toaddr, toid, &findnodeV4{
   		Target:     target,
   		Expiration: uint64(time.Now().Add(expiration).Unix()),
   	})
   	return nodes, <-rm.errc
   }
   ```

   

3. 查找3个随机的目标节点

   ```go
   for i := 0; i < 3; i++ {
   		tab.net.lookupRandom()
   	}
   ```

##### `doRevalidate`

doRevalidate检查随机存储桶中的最后一个节点是否仍然存在，如果不是，则替换或删除该节点。

主要以下几步：

1. 返回随机的非空K桶中的最后一个节点

   ```go
   last, bi := tab.nodeToRevalidate()
   ```

2. 对最后的节点执行Ping操作，然后等待Pong

   ```go
   remoteSeq, err := tab.net.ping(unwrapNode(last))
   ```

3. 如果节点ping通了的话，将节点移动到最前面

   ```go
   tab.bumpInBucket(b, last)
   ```

4. 没有收到回复，选择一个替换节点，或者如果没有任何替换节点，则删除该节点

   ```go
   tab.replace(b, last)
   ```

##### `copyLiveNodes`

copyLiveNodes将表中的节点添加到数据库,如果节点在表中的时间超过了5分钟。

这部分代码比较简单，就伸展阐述。

```go
if n.livenessChecks > 0 && now.Sub(n.addedAt) >= seedMinTableTime {
				tab.db.UpdateNode(unwrapNode(n))
			}
```

#### 3.检测各类信息

```go
go t.loop()
```

loop循环主要监听以下几类消息：

- case <-t.closeCtx.Done()：检测是否停止
- p := <-t.addReplyMatcher：检测是否有添加新的待处理消息
- r := <-t.gotreply：检测是否接收到其他节点的回复消息

#### 4. 处理UDP数据包

```go
go t.readLoop(cfg.Unhandled)
```

主要有以下两件事：

1. 循环接收其他节点发来的udp消息

   ```go
   nbytes, from, err := t.conn.ReadFromUDP(buf)
   ```

2. 处理接收到的UDP消息

   ```go
   t.handlePacket(from, buf[:nbytes])
   ```

接下来对这两个函数进行进一步的解析。

##### 接收UDP消息

接收UDP消息比较的简单，就是不断的从连接中读取Packet数据，它有以下几种消息：

- `ping`：用于判断远程节点是否在线。

- `pong`：用于回复`ping`消息的响应。

- `findnode`：查找与给定的目标节点相近的节点。

- `neighbors`：用于回复`findnode`的响应，与给定的目标节点相近的节点列表

-----

##### 处理UDP消息

主要做了以下几件事：

1. 数据包解码

   ```go
   packet, fromKey, hash, err := decodeV4(buf)
   ```

2. 检查数据包是否有效，是否可以处理

   ```go
    packet.preverify(t, from, fromID, fromKey)
   ```

   在校验这一块，涉及不同的消息类型不同的校验，我们来分别对各种消息进行分析。

   ①：`ping`

   - 校验消息是否过期
   - 校验公钥是否有效

   ②：`pong`

   - 校验消息是否过期
   - 校验回复是否正确

   ③：`findNodes`

   - 校验消息是否过期
   - 校验节点是否是最近的节点

   ④：`neighbors`

   - 校验消息是否过期
   - 用于回复`findnode`的响应，校验回复是否正确

3. 处理packet数据

   ```go
   packet.handle(t, from, fromID, hash)
   ```

   相同的，也会有4种消息，但是我们这边重点讲处理findNodes的消息：

   ```go
func (req *findnodeV4) handle(t *UDPv4, from *net.UDPAddr, fromID enode.ID, mac []byte) {
   ...
}
   ```
   
   我们这里就稍微介绍下如何处理`findnode`的消息：
   
   ```go
   func (req *findnodeV4) handle(t *UDPv4, from *net.UDPAddr, fromID enode.ID, mac []byte) {
   	// 确定最近的节点
   	target := enode.ID(crypto.Keccak256Hash(req.Target[:]))
   	t.tab.mutex.Lock()
   	//最接近的返回表中最接近给定id的n个节点
   	closest := t.tab.closest(target, bucketSize, true).entries
   	t.tab.mutex.Unlock()
   	// 以每个数据包最多maxNeighbors的块的形式发送邻居，以保持在数据包大小限制以下。
   	p := neighborsV4{Expiration: uint64(time.Now().Add(expiration).Unix())}
   	var sent bool
   	for _, n := range closest { //扫描这些最近的节点列表，然后一个包一个包的发送给对方
   		if netutil.CheckRelayIP(from.IP, n.IP()) == nil {
   			p.Nodes = append(p.Nodes, nodeToRPC(n))
   		}
   		if len(p.Nodes) == maxNeighbors {
   			t.send(from, fromID, &p)//给对方发送 neighborsPacket 包，里面包含节点列表
   			p.Nodes = p.Nodes[:0]
   			sent = true
   		}
   	}
   	if len(p.Nodes) > 0 || !sent {
   		t.send(from, fromID, &p)
   	}
   }
   ```
   
   首先先确定最近的节点，再一个包一个包的发给对方，并校验节点的IP，最后把有效的节点发送给请求方。

----

## 涉及的结构体：

### UDP

- conn ：接口，包括了从UDP中读取和写入，关闭UDP连接以及获取本地地址。
- netrestrict：IP网络列表
- localNode：本地节点
- tab：路由表

-------

### Table

- buckets：所有节点都加到这个里面，按照距离
- nursery：启动节点
- rand：随机来源
- ips：跟踪IP，确保IP中最多N个属于同一网络范围

- net: UDP 传输的接口
  - 返回本地节点
  - 将enrRequest发送到给定的节点并等待响应
  - findnode向给定节点发送一个findnode请求，并等待该节点最多发送了k个邻居
  - 返回查找最近的节点
  -  将ping消息发送到给定的节点，然后等待答复

以下是table的结构图：

![image-20201112104254003](https://tva1.sinaimg.cn/large/0081Kckwgy1gkm6yzncc3j30t00ggdim.jpg)

-------

## 思维导图

> [思维导图获取地址](https://github.com/blockchainGuide/blockchainguide/tree/main/%E4%B8%BB%E6%B5%81%E5%8C%BA%E5%9D%97%E9%93%BE/ethereum/%E4%BB%A5%E5%A4%AA%E5%9D%8A%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/p2p)

![image-20201123211034861](https://tva1.sinaimg.cn/large/0081Kckwgy1gkzexhy1kqj31d80mqafh.jpg)

## 参考文档

> http://mindcarver.cn/     ⭐️⭐️⭐️⭐️
>
> https://github.com/blockchainGuide/ ⭐️⭐️⭐️⭐️
>
> https://www.cnblogs.com/xiaolincoding/p/12571184.html 
>
> http://qjpcpu.github.io/blog/2018/01/29/shen-ru-ethereumyuan-ma-p2pmo-kuai-ji-chu-jie-gou/
>
> https://www.jianshu.com/p/b232c870dcd2
>
> https://bbs.huaweicloud.com/blogs/113684
>
> https://www.jianshu.com/p/94d02a41a146