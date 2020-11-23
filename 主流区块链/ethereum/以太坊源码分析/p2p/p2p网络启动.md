> 死磕以太坊源码分析之p2p网络启动

## p2p源码目录

```
discover/          基于UDP的节点发现V4协议
  discv5/            节点发现V5协议
  enode/             节点信息
  enr/               以太坊节点记录（ethereum node records)
  nat/               网络地址转换，用于内网穿透
  netutil/
  protocol/
  simulations/       本地p2p网络的模拟器
  dial.go            建立连接请求，以任务的形式 
  message.go         定义了读写的接口
  metrics.go         计时器和计量器工具
  peer.go            节点
  protocol.go        子协议
  rlpx.go            加密传输协议 
  server.go          底层p2p网络的函数入口 
```



## 启动p2p网络



启动p2p网络主要会做以下几件事：

1. 发现远程节点，建立相邻节点列表
2. 监听远程节点发过来的建立TCP请求
3. 向远程节点发送建立TCP连接请求

首先找到p2p网络启动的入口：

### Start()

`start`函数主要做了以下6件事：

1. 初始化server的字段
2. 设置本地节点setupLocalNode
3. 设置监听TCP连接请求setupListening
4. 设置节点发现（setupDiscovery）V4版本
5. 设置最大可以主动发起的连接为50/3
6. srv.run(dialer) 发起建立TCP连接请求

其中setupLocalNode、setupListening、setupDiscovery、newDialState、srv.run(dialer)是我们要重点分析的函数。

#### 设置本地节点

进入到setupLocalNode中：

①：创建devp2p握手

```GO
pubkey := crypto.FromECDSAPub(&srv.PrivateKey.PublicKey)
	srv.ourHandshake = &protoHandshake{Version: baseProtocolVersion, Name: srv.Name, ID: pubkey[1:]}
	for _, p := range srv.Protocols {
		srv.ourHandshake.Caps = append(srv.ourHandshake.Caps, p.cap())
	}
sort.Sort(capsByNameAndVersion(srv.ourHandshake.Caps))
```

握手协议包括协议版本号，节点名称和节点的公钥，存入到Caps中要根据名称和协议排序。

②：创建本地节点

```GO
db, err := enode.OpenDB(srv.Config.NodeDatabase)
	if err != nil {
		return err
	}
	srv.nodedb = db
	srv.localnode = enode.NewLocalNode(db, srv.PrivateKey)
	srv.localnode.SetFallbackIP(net.IP{127, 0, 0, 1})
	// TODO: check conflicts
	for _, p := range srv.Protocols {
		for _, e := range p.Attributes {
			srv.localnode.Set(e)
		}
	}
```

首先从节点数据库中去获取节点信息，如果不存在则新建本地节点并设置默认IP，同时将节点记录的协议特定信息存入到本地节点中。

-----

#### 设置监听

进入到`setupListening`:

①：启动监听器

②：如果配置了NAT，则更新本地节点记录并映射TCP监听端口

```GO
if tcp, ok := listener.Addr().(*net.TCPAddr); ok {
		srv.localnode.Set(enr.TCP(tcp.Port))
		if !tcp.IP.IsLoopback() && srv.NAT != nil {
			srv.loopWG.Add(1)
			go func() {
				nat.Map(srv.NAT, srv.quit, "tcp", tcp.Port, tcp.Port, "ethereum p2p")
				srv.loopWG.Done()
			}()
		}
	}
```

③：开启P2P监听，接收`inbound`连接

```GO
srv.listenLoop()
```

这个函数需要进一步分析：

主要有以下逻辑：

- 首先`defaultMaxPendingPeers`这个字段指的是`inbound` 和`outbound`连接，默认最大值为50

- 将监听的连接返回给`listener`

  ```go
  fd, err = srv.listener.Accept()
  ```

- 获取监听的连接的地址并检查这个连接

  ```GO
  remoteIP := netutil.AddrIP(fd.RemoteAddr())
  if err := srv.checkInboundConn(fd, remoteIP); err != nil {
    .....
  }
  ```

  `checkInboundConn`主要是做了以下的判断：

  - 拒绝不符合NetRestrict的连接（NetRestrict是指已经限定了某些连接，除此之外会拒绝）
  - 拒绝尝试过多的节点

- 最后真正建立连接

  ```GO
  go func() {
  			srv.SetupConn(fd, inboundConn, nil)// 连接建立过程（将连接添加为peer）
  			slots <- struct{}{}
  		}()
  ```

  要注意setupConn的第三个字段传入的是nil，表示还没有拨号，如果正在拨号的话需要节点公钥。

  ```GO
  var dialPubkey *ecdsa.PublicKey
  	if dialDest != nil {
  		dialPubkey = new(ecdsa.PublicKey)
  		if err := dialDest.Load((*enode.Secp256k1)(dialPubkey)); err != nil {
  			return errors.New("dial destination doesn't have a secp256k1 public key")
  		}
  	}
  ```

  之后就是进行RLPX（RLPX会单独讲）握手

  ```go
  remotePubkey, err := c.doEncHandshake(srv.PrivateKey, dialPubkey)
  ```

  如果dialDest 不为nil，检查公钥是否匹配，如果为nil,就从连接中返回一个node出来

  ```GO
  if dialDest != nil {
  		// For dialed connections, check that the remote public key matches.
  		//对于拨号连接，请检查远程公钥是否匹配
  		if dialPubkey.X.Cmp(remotePubkey.X) != 0 || dialPubkey.Y.Cmp(remotePubkey.Y) != 0 {
  			return DiscUnexpectedIdentity
  		}
  		c.node = dialDest
  	} else {
  		c.node = nodeFromConn(remotePubkey, c.fd)
  	}
  ```

  接下来就是真正执行握手了 ,这部分也属于RLPX，跳过

  ```GO
  phs, err := c.doProtoHandshake(srv.ourHandshake)
  ```

  之后要进行检查，如果成功了的话，连接就会作为节点被添加，并且启动了runPeer.

  到此为止，整个listenLoop 就完成了。

  -----

#### 设置节点发现

进入到`srv.setupDiscovery()`

①：添加特定于协议的发现源

```GO
added := make(map[string]bool)
	for _, proto := range srv.Protocols {
		if proto.DialCandidates != nil && !added[proto.Name] {
			srv.discmix.AddSource(proto.DialCandidates)
			added[proto.Name] = true
		}
	}
```

②：如果DHT禁用的话，就不要在UDP上监听

```go
if srv.NoDiscovery && !srv.DiscoveryV5 {
		return nil
	}
```

③：监听给定的socket 上的发现的包

```go
ntab, err := discover.ListenUDP(conn, srv.localnode, cfg)
```

#### 创建DialState

dialstate负责拨号和查找发现。

①：初始化dialstate

```go
s := &dialstate{
   maxDynDials: maxdyn,
   self:        self,
   netrestrict: cfg.NetRestrict,
   log:         cfg.Logger,
   static:      make(map[enode.ID]*dialTask),
   dialing:     make(map[enode.ID]connFlag),
   bootnodes:   make([]*enode.Node, len(cfg.BootstrapNodes)),
}
```

②：加入初始引导节点

```GO
	copy(s.bootnodes, cfg.BootstrapNodes)
	if s.log == nil {
		s.log = log.Root()
	}
```

③： 加入静态节点

```GO
for _, n := range cfg.StaticNodes {
		s.addStatic(n)
	}
```

`bootnodes`是初始引导节点，在节点没有接收到任何节点的连接请求，也没有节点可以给我们邻居节点的时候，就去连接`bootnodes`，它硬编码在了以太坊的源码中。

`static`是静态节点，如果我们想和某些节点保持长期的连接，就把它们加入到静态节点的列表中

接下来就是到了运行p2p网络的时候了，主要的函数是：`go srv.run(dialer)`

-----

## 运行p2p网络

### srv.run(dialer)

在p2p网络启动时候，我们会监听远程节点发送过来的TCP请求，到了运行p2p网络的时候，我们则会向远程节点发起TCP的连接请求。首先我们要知道我们所说的发起TCP连接请求可以形容成拨号，每个拨号都是以任务的形式存在，进入到`srv.run(dialer)`分析

整个函数就是一个循环，介绍下它的主要功能：

#### 发起TCP连接任务

```GO
scheduleTasks()
```

`scheduleTasks`主要是从queued task 中去获取任务，通过查询dialer以查找新任务并立即启动尽可能多的任务，我们这里要注意个变量`maxActiveDialTasks`,它的默认值为16 ，而安排任务的核心方法是：

```go
nt := dialstate.newTasks(len(runningTasks)+len(queuedTasks), peers, time.Now())
```

主要做了以下几件事：

①：为没有连接的静态节点创建拨号任务

```go
for id, t := range s.static {
		err := s.checkDial(t.dest, peers)
		switch err {
		case errNotWhitelisted, errSelf:
			s.log.Warn("Removing static dial candidate", "id", t.dest.ID, "addr", &net.TCPAddr{IP: t.dest.IP(), Port: t.dest.TCP()}, "err", err)
			delete(s.static, t.dest.ID())
		case nil:
			s.dialing[id] = t.flags
			newtasks = append(newtasks, t)
		}
	}
```

首先对拨号节点进行校验：正在连接，已经连接，是本身，不在白名单中，最近连接过的都会报错，并且不是在白名单中的和自身的节点会直接从静态节点列表中删除，校验通过的创建任务。

②：计算所需的动态拨号数

```GO
needDynDials := s.maxDynDials
	for _, p := range peers {
		if p.rw.is(dynDialedConn) {
			needDynDials--
		}
	}
	for _, flag := range s.dialing {
		if flag&dynDialedConn != 0 {
			needDynDials--
		}
	}
```

我们主动发起的TCP连接请求是由节点最大连接数除以拨号比率得出的，即`maxPeers/radio`，同时我们会判断节点中是否已经有建立了连接的节点和正在拨号的节点，有的话会needDynDials会减去。

③：如果找不到任何的peers,就去随机找bootnode，发起连接

不过这个一般适用在**测试网或者私链**。

```GO
if len(peers) == 0 && len(s.bootnodes) > 0 && needDynDials > 0 && now.Sub(s.start) > fallbackInterval {
		bootnode := s.bootnodes[0]
		s.bootnodes = append(s.bootnodes[:0], s.bootnodes[1:]...)
		s.bootnodes = append(s.bootnodes, bootnode)
		if addDial(dynDialedConn, bootnode) {
			needDynDials--
		}
	}
```

④：从节点发现结果中创建动态拨号任务

如果不满足最大任务数量的话，就去`s.lookupBuf`中寻找，`lookupBuf`通过KAD算法获取的节点。

```go
for ; i < len(s.lookupBuf) && needDynDials > 0; i++ {
		if addDial(dynDialedConn, s.lookupBuf[i]) {
			needDynDials--
		}
	}
	s.lookupBuf = s.lookupBuf[:copy(s.lookupBuf, s.lookupBuf[i:])]

if len(s.lookupBuf) < needDynDials && !s.lookupRunning {
		s.lookupRunning = true
		newtasks = append(newtasks, &discoverTask{want: needDynDials - len(s.lookupBuf)})
	}
```

⑤：没有需要执行的任务，保持拨号逻辑继续运行

```GO
if nRunning == 0 && len(newtasks) == 0 && s.hist.Len() > 0 {
		t := &waitExpireTask{s.hist.nextExpiry().Sub(now)}
		newtasks = append(newtasks, t)
	}
```

到此创建新任务结束，返回`newTasks`

-----

#### 执行TCP连接任务

直到满足最大活动任务数才开始任务执行，具体的执行过程在以下代码：

```go
startTasks := func(ts []task) (rest []task) {
		i := 0
		for ; len(runningTasks) < maxActiveDialTasks && i < len(ts); i++ {
			t := ts[i]
			srv.log.Trace("New dial task", "task", t)
			go func() { t.Do(srv); taskdone <- t }()
			runningTasks = append(runningTasks, t)
		}
		return ts[i:]
	}
```

```go
t.Do(srv);
```

执行的主要任务包括下面几种：

1. dialTask
2. discoverTask
3. waitExpireTask

最关键的就是`dialTask`

```go
func (t *dialTask) Do(srv *Server) {
	if t.dest.Incomplete() {
		if !t.resolve(srv) {
			return
		}
	}
	err := t.dial(srv, t.dest)
	if err != nil {
		srv.log.Trace("Dial error", "task", t, "err", err)
		// Try resolving the ID of static nodes if dialing failed.
		if _, ok := err.(*dialError); ok && t.flags&staticDialedConn != 0 {
			if t.resolve(srv) {
				t.dial(srv, t.dest)
			}
		}
	}
}
```

真正的连接是在`t.dail`中做的：

```go
// 实际的网络连接操作
func (t *dialTask) dial(srv *Server, dest *enode.Node) error {
	fd, err := srv.Dialer.Dial(dest)
	if err != nil {
		return &dialError{err}
	}
	mfd := newMeteredConn(fd, false, &net.TCPAddr{IP: dest.IP(), Port: dest.TCP()})
	return srv.SetupConn(mfd, t.flags, dest)
}
```

再往下面就没必要深究了，实际的网络连接操作到此为止了。

------

#### 管理TCP连接任务

在TCP连接任务完成后，会对连接有各种处理，如下：

①：停止p2p服务

```GO
case <-srv.quit:
break running
```

②：添加静态节点到peer列表

```go
case n := <-srv.addstatic:
srv.log.Trace("Adding static node", "node", n)
dialstate.addStatic(n)
```

③：发送断开连接请求，并断开连接

```go
case n := <-srv.removestatic:
dialstate.removeStatic(n)
			if p, ok := peers[n.ID()]; ok {
				p.Disconnect(DiscRequested)
			}
```

断开连接会立即返回，并且不会等连接关闭。

④：标记可信节点

```go
case n := <-srv.addtrusted:
trusted[n.ID()] = true
```

⑤：从信任节点中删除一个节点

```go
case n := <-srv.removetrusted:
delete(trusted, n.ID())
```

⑥：拨号任务完成

```go
case t := <-taskdone:newTasks
dialstate.taskDone(t, time.Now())
delTask(t)
```

⑦：连接已通过加密握手,远程身份是已知的（但尚未经过验证）

```go
case c := <-srv.checkpointPostHandshake:
c.cont <- srv.postHandshakeChecks(peers, inboundCount, c)
```

⑧：连接已通过协议握手，已知其功能并验证了远程身份

```go
err := srv.addPeerChecks(peers, inboundCount, c)
			if err == nil {
			// 握手完成，所有检查完毕
				p := newPeer(srv.log, c, srv.Protocols)
			//启用了消息事件就把peerfeed传给peer
				if srv.EnableMsgEvents {
					p.events = &srv.peerFeed
				}
				name := truncateName(c.name)
	p.RemoteAddr(), "peers", len(peers)+1, "name", name)
				go srv.runPeer(p) // 重点
				peers[c.node.ID()] = p
				if p.Inbound() {
					inboundCount++
				}
				if conn, ok := c.fd.(*meteredConn); ok {
					conn.handshakeDone(p)// 
				}
			}
```

`addPeerChecks`会删除没有匹配协议的连接，并且会重复握手后检查，因为自执行这些检查后可能已更改。连接通过握手后，将调用`handshakeDone`

⑨：Peer断开连接

```go
case pd := <-srv.delpeer:
d := common.PrettyDuration(mclock.Now() - pd.created)
			pd.log.Debug("Removing p2p peer", "addr", pd.RemoteAddr(), "peers", len(peers)-1, "duration", d, "req", pd.requested, "err", pd.err)
			delete(peers, pd.ID())
			if pd.Inbound() {
				inboundCount--
			}
```

到此为止整个主要的处理TCP连接的循环讲解结束。

-----

## 总结

1. 开启p2p网络主要包括：设置本地节点，监听TCP连接以及设置节点发现
2. 运行P2P网络之后主要包括：发起TCP连接并执行连接，以及相关的连接处理。

## 参考

> https://github.com/blockchainGuide/   很全面的区块链开源学习资源