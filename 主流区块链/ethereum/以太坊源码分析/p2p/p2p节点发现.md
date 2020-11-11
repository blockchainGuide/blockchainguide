

以太坊的节点发现分为v4和v5版本，目前主要套路v5版本。



## 1.发现要连接的RLPX节点

## 2.基于UDP的RPC协议

## 定义4种类型ping,pong,findnode,neighours



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

`discover.ListenUDP`方法即开启了节点发现的功能.进入到`ListenUDP`:

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

	tab, err := newTable(t, ln.Database(), cfg.Bootnodes, t.log)
	if err != nil {
		return nil, err
	}
	t.tab = tab
	go tab.loop()

	t.wg.Add(2)
	go t.loop()
	go t.readLoop(cfg.Unhandled)
	return t, nil
}
```



主要做了以下几件事：

#### 1.新建路由表

新建路由表做了以下几件事：

- 初始化table对象
- setFallbackNodes：设置初始接触点。如果表为空，并且数据库中没有已知的节点，则这些节点用于连接到网络
- tab.seedRand：使用提供的种子值将生成器初始化为确定性状态
- loadSeedNodes：加载种子节点；从保留已知节点的数据库中随机的抽取30个节点，再加上引导节点列表中的节点，放置入k桶中，如果K桶没有空间，则假如到替换列表中。



#### 2.刷新K桶

定时运行`doRefresh`、`doRevalidate`、`copyLiveNodes`进行刷新K桶。

以太坊的k桶设置：
`alpha` ：3
`nBuckets`：k桶数量为17
`bucketSize`：k桶中最多存16个节点
`maxReplacements`：每个k桶的候选节点列表最多存10个节点

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

loop循环主要监听以下几类消息：

- case <-t.closeCtx.Done()：检测是否停止
- p := <-t.addReplyMatcher：检测是否有添加新的待处理消息
- r := <-t.gotreply：检测是否接收到其他节点的回复消息

#### 4. 处理UDP数据包

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
- `enrRequest`:
- `enrResponse`:

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

   ①：处理`ping`消息

   ②：处理`pong`消息

   ③：处理`findNodes`消息

   ④：处理`neighbors`消息

---



## 涉及的结构体：

### UDP

- conn ：接口，包括了从UDP中读取和写入，关闭UDP连接以及获取本地地址。
- netrestrict：IP网络列表
- localNode：本地节点
- tab：路由表



### Table

- buckets：所有节点都加到这个里面，按照距离
- nursery：启动节点
- rand：随机来源，定期播种
- ips：跟踪IP，确保IP中最多N个属于同一网络范围

- net: UDP 传输的接口
  - 返回本地节点
  - 将enrRequest发送到给定的节点并等待响应
  - findnode向给定节点发送一个findnode请求，并等待该节点最多发送了k个邻居
  - 返回查找最近的节点
  -  将ping消息发送到给定的节点，然后等待答复
- 



-----

## 参考文档

> https://www.cnblogs.com/xiaolincoding/p/12571184.html (ping 工作原理)