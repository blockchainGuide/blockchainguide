> 死磕hyperledger fabric源码|Peer节点启动
>
> 文章及代码：https://github.com/blockchainGuide/
>
> 分支：v1.1.0

![b43356d6533644cca27da75d63f275fa](https://tva1.sinaimg.cn/large/008eGmZEgy1gn1cycj35uj31c00u0gww.jpg)

## 启动流程概述

入口：peer/main.go:

`main`()函数负责初始化`peer`主命令对象，注册子命令与初始化环境配置，解析用户输入子命令`start`并启动Peer节点，包括如下流程步骤：

- 定义、注册命令与初始化基本配置。基于`Cobra`组件定义peer主命令对象`mainCmd`，并通过`Viper`组件调用`InitConfig`()函数，从本地`core.yaml`配置文件、环境变量、命令行选项等读取与解析`peer`命令的相关配置。同时，初始化主命令`mainCmd`的标志位选项 `version`、`logging-level`等，然后在主命令`mainCmd`上注册`version、node、chaincode、channel`等子命令，设置最大可用`CPU`核数与日志后端； 
- 初始化本地`MSP`组件。通过`Viper`组件获取`MSP`组件的配置文件路径`mspMgrConfigDir`、`BCCSP`配置项`bccspConfig`、MSP名称ID即localMSPID、`MSP`组件类型`localMSPType`等，基于这4个参数构造本地`MSP`配置对象，接着创建默认的`bccspmsp`结构对象作为本地`MSP`组件，并解析`MSP`配置对象与初始化本地`MSP`组件； 
- 执行mainCmd.Execute()方法启动`Peer`节点

接下来将会分别对这几个关键部分进行细说。



## 定义、注册命令与初始化基本配置

### 定义主命令

代码分析 如下：

```go
var mainCmd = &cobra.Command{ // 基于Cobra组件构造主命令
	Use: "peer", // 定义命令使用方法
	//// 定义执行函数
	PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
		//检查CORE_LOGGING_LEVEL环境变量，覆盖所有其他日志设置值。否则，使用core.yaml文件中的配置值
		var loggingSpec string
		if viper.GetString("logging_level") != "" {
			loggingSpec = viper.GetString("logging_level") // 获取配置文件中的日志级别
		} else {
			loggingSpec = viper.GetString("logging.level") // 获取配置文件中的日志级别
		}
		flogging.InitFromSpec(loggingSpec) // 根据配置的日志级别初始化日志记录器
		return nil
	},
	Run: func(cmd *cobra.Command, args []string) { // 定义执行函数
		if versionFlag {
			fmt.Print(version.GetInfo()) // 打印peer程序版本信息
		} else {
			cmd.HelpFunc()(cmd, args) // 直接打印命令帮助信息
		}
	},
```

### 注册子命令

将几类子命令注册到主命令上，种类如下：

- channel通道子命令：用于创建应用通道、获取区块、Peer节点加入应用通道、获取节点所加入的应用通道列表、更新应用通道配置、签名配置交易文件、获取指定的应用通道信息等，包括`create、fetch、join、list、update、signconfigtx、getinfo`等子命令；
- chaincode链码子命令：用于安装链码、实例化（部署）链码、调用链码、打包链码、查询链码、签名链码包、升级链码、获取通道链码列表等，包括`install、instantiate、invoke、package、query、signpackage、upgrade、list`等子命令； 
- node节点子命令：用于管理节点服务进程与查询服务状态，包括`start、status`等子命令；
- logging日志子命令：用于获取、设置与恢复日志级别功能，包括`getlevel、setlevel、 revertlevels`等子命令；
- version版本子命令：用于打印`Fabric`中的`Peer`节点服务器版本信息。 

```go
viper.SetEnvPrefix(cmdRoot)               // 设置环境变量前缀core
	viper.AutomaticEnv()                      // 查找匹配环境变量
	replacer := strings.NewReplacer(".", "_") // 创建替换符
	viper.SetEnvKeyReplacer(replacer)         // 设置环境变量替换符
	// 定义命令行选项集合，对所有peer及其子命令都有效
	mainFlags := mainCmd.PersistentFlags()
	// 设置绑定version与logging-level选项
	mainFlags.BoolVarP(&versionFlag, "version", "v", false, "Display current version of fabric peer server")
	mainFlags.String("logging-level", "", "Default logging level and overrides, see core.yaml for full syntax")
	// Viper配置绑定命令行选项
	viper.BindPFlag("logging_level", mainFlags.Lookup("logging-level"))
	// 注册子命令
	mainCmd.AddCommand(version.Cmd())       // version子命令
	mainCmd.AddCommand(node.Cmd())          // node子命令start、status
	mainCmd.AddCommand(chaincode.Cmd(nil))  // chaincode子命令install等
	mainCmd.AddCommand(clilogging.Cmd(nil)) // cli日志子命令 getlevel
	mainCmd.AddCommand(channel.Cmd(nil))    // channel子命令 create等
	// 加载配置文件core.yaml
	err := common.InitConfig(cmdRoot)
	...
	runtime.GOMAXPROCS(viper.GetInt("peer.gomaxprocs")) // 设置最大可用的CPU核数
	// setup system-wide logging backend based on settings from core.yaml
	// 初始化系统日志后端
	flogging.InitBackend(flogging.SetFormat(viper.GetString("logging.format")), logOutput)
```

## 初始化本地MSP组件 

`MSP`组件是管理本地成员身份的重要安全模块，封装了根`CA`证书、本地签名者实体等.

```go
// 初始化本地MSP组件对象
	var mspMgrConfigDir = config.GetPath("peer.mspConfigPath") // 获取MSP配置文件路径
	var mspID = viper.GetString("peer.localMspId")             // 获取本地MSP名称
	var mspType = viper.GetString("peer.localMspType")         // 获取本地MSP组件类型
	if mspType == "" {
		// 默认设置MSP组件类型为FABRIC类型
		mspType = msp.ProviderTypeToString(msp.FABRIC)
	}
	// 获取BCCSP组件配置信息，初始化MSP组件对象
	err = common.InitCrypto(mspMgrConfigDir, mspID, mspType)
```



## 执行主命令

函数如下：

```go
if mainCmd.Execute() != nil {}
```

接下来进入到`Execute()`函数中继续分析： /vendor/github.com/spf13/cobra/command.go/ExecuteC()

```go
func (c *Command) ExecuteC() (cmd *Command, err error) {}
```

通过`Cobra`组件调用主命令`Execute`()方法，执行`peer node start`命令启动`Peer`节点。其中，`Cobra`组件解析完用户输入的命令行选项之后，依次执行节点启动命令nodeStartCmd对象中定义的所有相关的执行方法，并按照`cobra.Command`命令中定义的如下顺序来执行

- PersistentPreRunE()/PersistentPreRun()； 
- PreRunE()/PreRun()； 
- RunE()/Run()； 
- PostRunE()/PostRun()； 
- PersistentPostRunE()/PersistentPostRun();

到这里为止节点命令开启执行，因为这部分主要是讲的节点启动，所以下面集中将节点启动命令执行的运行流程。

## 节点启动命令执行

节点启动的命令可以根据以下代码路径查找：

```go
mainCmd.AddCommand(node.Cmd()) 
```

```go
func Cmd() *cobra.Command {
	nodeCmd.AddCommand(startCmd())
	nodeCmd.AddCommand(statusCmd())
	return nodeCmd
}
```

这里只讨论节点启动命令

```go
func startCmd() *cobra.Command {
...
	return nodeStartCmd
}
```

```go
var nodeStartCmd = &cobra.Command{
...
		return serve(args)
	},
}
```

正式进入到serve函数讨论：

### 初始化资源

①：*获取本地MSP组件类型并检查MSP组件类型*

目前，`Hyperledger Fabric`支持`FABRIC`类型和`IDEMIX`类型两种`MSP`组件，默认采用基于`BCCSP`组件构建的`FABRIC`类型`MSP`组件.

```go
mspType := mgmt.GetLocalMSP().GetType() // 获取本地MSP组件类型
	if mspType != msp.FABRIC {              // 检查MSP组件类型
		panic("Unsupported msp type " + msp.ProviderTypeToString(mspType))
	}
```

②：*初始化资源访问策略提供者*

```GO
aclmgmt.RegisterACLProvider(nil)
```

③：*初始化本地账本管理器*

```go
ledgermgmt.Initialize(peer.ConfigTxProcessors) 	
```

> core/ledger/ledgermgmt/ledger_mgmt.go/initialize

```go
func initialize(customTxProcessors customtx.Processors) {
	logger.Info("Initializing ledger mgmt")
	lock.Lock()
	defer lock.Unlock() // 设置Peer节点初始化标志位为true
	initialized = true
	// 创建已打开的账本字典openedLedgers
	openedLedgers = make(map[string]ledger.PeerLedger)
	// 初始化配置交易消息处理器字典，设置给全局变量processors字典
	customtx.Initialize(customTxProcessors)
	cceventmgmt.Initialize()                // 初始化链码事件管理器
	provider, err := kvledger.NewProvider() // 创建本地Peer节点账本提供者
	if err != nil {
		panic(fmt.Errorf("Error in instantiating ledger provider: %s", err))
	}
	provider.Initialize(kvLedgerStateListeners) // 初始化状态监听器
	ledgerProvider = provider                   // 设置为全局默认的Peer节点账本提供者
	logger.Info("ledger mgmt initialized")
}
```

正本提供者有以下几种：

- 账本ID数据库（idStore类型）：提供存储账本ID（即链ID）与创世区块键值对的`LevelDB`数据库； 
- 账本数据存储对象提供者（ledgerstorage.Provider类型）：创建账本数据存储对象，负责管理区块数据文件、隐私数据库、区块索引数据库等； 

- 历史数据库提供者（HistoryDBProvider类型）：创建历史数据库，存储每个状态数据的历史信息； 

- 状态数据库提供者（CommonStorageDBProvider类型）：创建状态数据库（`LevelDB`或`CouchDB`类型），存储世界状态（`world state`），包括有效交易的公有数据与隐私数据。

④：*初始化服务器参数*

```go
if chaincodeDevMode {
		//设置链码模式
		viper.Set("chaincode.mode", chaincode.DevModeUserRunsChaincode)

	}
	// 读取配置并缓存Peer节点地址与端点
	if err := peer.CacheConfiguration(); err != nil {
		return err
	}
	// 获取缓存的Peer端点
	peerEndpoint, err := peer.GetPeerEndpoint()
	...
	var peerHost string
	// 获取Peer节点IP地址，注意IP地址与端口已经被分离
	peerHost, _, err = net.SplitHostPort(peerEndpoint.Address)
```

### 创建GRPC服务器

①：*创建gRPC服务器*

`serve()`函数创建了至少 3 个`gRPC`服务器（独立端口），用于注册`Peer`节点功能服务器，如下所示：

| 序号 | 端口 | 功能服务器                     | 说明                           | 服务接口          |
| ---- | ---- | ------------------------------ | ------------------------------ | ----------------- |
| 1    | 7051 | DeliverEvents事件服务器        | 处理区块请求消息               | Deliver()         |
|      | 7051 | Admin服务器                    | 获取节点状态、维护日志等       | GetStatus()       |
|      | 7051 | Endorser背书服务器             | 提供背书服务                   | ProcessProposal() |
|      | 7051 | Gossip消息服务器               | 组织内节点分发数据与同步状态等 | GossipStream()    |
| 2    | 7052 | chaincodeSupport链码支持服务器 | 提供Peer节点链码支持服务       | Register()        |
| 3    | 7053 | EventHub事件服务器             | 提供订阅事件服务(1.3.0废弃)    | Chat()            |

```go
serverConfig, err := peer.GetServerConfig()
	...
peerServer, err := peer.CreatePeerServer(listenAddr, serverConfig)
```

②：*创建EventHub事件服务器*

```go
	if serverConfig.SecOpts.UseTLS {
		...
		cs := comm.GetCredentialSupport() // 创建证书支持对象CredentialSupport结构对象
		cs.ServerRootCAs = serverConfig.SecOpts.ServerRootCAs
		//// 获取gRPC客户端证书用于TLS连接认证
		clientCert, err := peer.GetClientCertificate()
		// 设置客户端证书
		comm.GetCredentialSupport().SetClientCertificate(clientCert)
	}
	//// 创建事件EventHub服务器（7053端口）
	ehubGrpcServer, err := createEventHubServer(serverConfig)
```

③：*创建DeliverEvents事件服务器*

`serve()`函数检查如果开启了双向的`TLS`安全认证，则设置`mutualTLS`标志位为`true`，并定义获取资源策略检查器即`policyCheckerProvider`()函数。该函数将直接调用全局变量`aclProvider`对象的`CheckACL`()方法，检查签名消息在通道（channelID）上是否满足指定资源的访问控制权限策略。 

接着，`serve`()函数调用`peer.NewDeliverEventsServer()`函数，基于`mutualTLS`、`policy-CheckerProvider`等参数创建`DeliverEvents`事件服务器`abServer`，提供`Deliver()`与`DeliverFiltered`()服务接口，分别用于处理请求正常区块与过滤区块的消息。

然后调用`pb.RegisterDeliverServer()`方法，将`DeliverEvents`事件服务器`abServer`注册到默认的`gRPC`服务器上（7051端口），以提供本地事件服务。

```go
//// 检查是否开启了双向的TLS安全认证
	mutualTLS := serverConfig.SecOpts.UseTLS && serverConfig.SecOpts.RequireClientCert
	//// 定义资源访问权限策略检查函数
	policyCheckerProvider := func(resourceName string) deliver.PolicyChecker {
		return func(env *cb.Envelope, channelID string) error {
			return aclmgmt.GetACLProvider().CheckACL(resourceName, channelID, env)
		}
	}
	//创建DeliverEvents事件服务器，并注册到Peer节点gRPC服务器上（7051端口）
	abServer := peer.NewDeliverEventsServer(mutualTLS, policyCheckerProvider, &peer.DeliverSupportManager{})
	pb.RegisterDeliverServer(peerServer.Server(), abServer)
```

④：*创建ChaincodeSupport链码支持服务器*

```go
//创建链码支持服务专用gRPC服务器与链码支持服务实例ChaincodeSupport（专用端口或7052端口）
	ccSrv, ccEndpoint, err := createChaincodeServer(ca, peerHost)
	if err != nil {
		logger.Panicf("Failed to create chaincode server: %s", err)
	}
	//将链码支持服务器实例ChaincodeSupoort对象注册到Peer节点gRPC服务器上
	// 同时注册系统链码以支持部署调用系统链码
	registerChaincodeSupport(ccSrv, ccEndpoint, ca)
	go ccSrv.Start() //启动gRPC服务器提供链码支持服务
```

⑤：*创建Admin管理服务器与Endorser背书服务器*

```go
	//创建Admin管理服务器与
	pb.RegisterAdminServer(peerServer.Server(), core.NewAdminServer())
	// 定义Gossip协议分发隐私数据函数
	privDataDist := func(channel string, txID string, privateData *rwset.TxPvtReadWriteSet) error {
		return service.GetGossipService().DistributePrivateData(channel, txID, privateData)
	}
	//创建新的EndorserServer背书节点服务器
	serverEndorser := endorser.NewEndorserServer(privDataDist, &endorser.SupportImpl{})
	libConf := library.Config{}
	if err = viperutil.EnhancedExactUnmarshalKey("peer.handlers", &libConf); err != nil {
		return errors.WithMessage(err, "could not load YAML config")
	}
	//// 创建消息过滤器列表
	authFilters := library.InitRegistry(libConf).Lookup(library.Auth).([]authHandler.Filter)
	// 将所有消息过滤器均构造成消息过滤器链，并返回第1个过滤器（Filter类型，实现了EndorserServer // 接口）
	auth := authHandler.ChainFilters(serverEndorser, authFilters...)
	// Register the Endorser server
	//// 注册EndorserServer背书服务器到gRPC服务器
	pb.RegisterEndorserServer(peerServer.Server(), auth)
```

⑥：*创建Gossip消息服务器*  

```go
	//获取Bootstrap连接的初始节点地址列表，默认为127.0.0.1:7051
	bootstrap := viper.GetStringSlice("peer.gossip.bootstrap")

	////获取本地MSP签名者身份实体 并序列化
	serializedIdentity, err := mgmt.GetLocalSigningIdentityOrPanic().Serialize()
	...
	messageCryptoService := peergossip.NewMCS( // 构造Gossip消息加密服务组件
		peer.NewChannelPolicyManagerGetter(), // 通道策略管理器获取组件
		localmsp.NewSigner(),                 // 本地签名者
		mgmt.NewDeserializersManager())       // 身份反序列化组件管理器
	secAdv := peergossip.NewSecurityAdvisor(mgmt.NewDeserializersManager())

	// callback function for secure dial options for gossip service
	//定义Gossip服务器回调函数，用于创建Gossip服务器安全配置的gRPC拨号连接选项
	secureDialOpts := func() []grpc.DialOption {
		var dialOpts []grpc.DialOption
		// set max send/recv msg sizes
		dialOpts = append(dialOpts, grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(comm.MaxRecvMsgSize()),
			grpc.MaxCallSendMsgSize(comm.MaxSendMsgSize()))) // 设置最大发送和接收消息字节数
		// set the keepalive options
		kaOpts := comm.DefaultKeepaliveOptions() // 获取默认的心跳消息keepalive选项
		...
		//在gRPC通信拨号连接选项中设置心跳通信keepalive选项
		dialOpts = append(dialOpts, comm.ClientKeepaliveOptions(kaOpts)...)

		if comm.TLSEnabled() { // 启用TLS安全认证，设置客户端TLS通信证书
			dialOpts = append(dialOpts, grpc.WithTransportCredentials(comm.GetCredentialSupport().GetPeerCredentials()))
		} else {
			dialOpts = append(dialOpts, grpc.WithInsecure()) // 否则，关闭TLS安全认证
		}
		return dialOpts
	}

	// 检查gRPC服务器端是否启用TLS安全认证，获取并设置服务器端与客户端身份证书
	var certs *common2.TLSCertificates
	if peerServer.TLSEnabled() {
		serverCert := peerServer.ServerCertificate()
		clientCert, err := peer.GetClientCertificate()
		if err != nil {
			return errors.Wrap(err, "failed obtaining client certificates")
		}
		certs = &common2.TLSCertificates{}
		certs.TLSServerCert.Store(&serverCert)
		certs.TLSClientCert.Store(&clientCert)
	}

	// 创建Gossip消息服务器实例gossipServiceInstance
	err = service.InitGossipService(serializedIdentity, peerEndpoint.Address, peerServer.Server(), certs,
		messageCryptoService, secAdv, secureDialOpts, bootstrap...)
```

### 部署系统链码与初始化现存通道的链结构 

①：*部署系统链码*

```go
	//部署系统链码
	initSysCCs()
```

②：*初始化现存通道上的链结构*

```go
//初始化现存通道上的链结构
	peer.Initialize(func(cid string) {
	})
```

> core/peer/peer.go/Initialize

```go
func Initialize(init func(string)) {
	nWorkers := viper.GetInt("peer.validatorPoolSize") // 获取交易验证线程数量
	if nWorkers <= 0 {
		nWorkers = runtime.NumCPU()
	}
	//// 设置信号量并发访问数量
	validationWorkersSemaphore = semaphore.NewWeighted(int64(nWorkers))

	chainInitializer = init // 设置初始化函数

	var cb *common.Block
	var ledger ledger.PeerLedger
	//// 初始化账本管理器
	ledgermgmt.Initialize(ConfigTxProcessors)
	//// 获取当前账本管理器下的账本ID列表
	ledgerIds, err := ledgermgmt.GetLedgerIDs()
	if err != nil {
		panic(fmt.Errorf("Error in initializing ledgermgmt: %s", err))
	}
	for _, cid := range ledgerIds {
		peerLogger.Infof("Loading chain %s", cid)
		//创建本地Peer节点账本
		if ledger, err = ledgermgmt.OpenLedger(cid); err != nil {
			peerLogger.Warningf("Failed to load ledger %s(%s)", cid, err)
			peerLogger.Debugf("Error while loading ledger %s with message %s. We continue to the next ledger rather than abort.", cid, err)
			continue
		}
		//// 从指定通道账本中获取最新配置区块
		if cb, err = getCurrConfigBlockFromLedger(ledger); err != nil {
			peerLogger.Warningf("Failed to find config block on ledger %s(%s)", cid, err)
			peerLogger.Debugf("Error while looking for config block on ledger %s with message %s. We continue to the next ledger rather than abort.", cid, err)
			continue
		}
		// Create a chain if we get a valid ledger with config block
		//// 在Peer节点上创建指定通道的链结构
		if err = createChain(cid, ledger, cb); err != nil {
			peerLogger.Warningf("Failed to load chain %s(%s)", cid, err)
			peerLogger.Debugf("Error reloading chain %s with message %s. We continue to the next chain rather than abort.", cid, err)
			continue
		}
		// 用自定义函数初始化通道链结构，如部署系统链码
		InitChain(cid)
	}
}
```

### 启动gRPC服务器与profile服务器

```go
// 建立传递错误消息的通道
	serve := make(chan error)
	// 传递信号的通道
	sigs := make(chan os.Signal, 1)
	// 设置本进程信号通道的通知信号，包括中断/终止信号
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	go func() {// 设置本进程阻塞等待的特定通知信号
		sig := <-sigs // 从sigs通道读取信号值，阻塞等待方式
		logger.Debugf("sig: %s", sig)
		serve <- nil
	}()
		// 利用goroutine 启动gRPC服务器（7051端口，注册了Admin管理服务器、Endorser背书服务器等）
	go func() {
		var grpcErr error
		if grpcErr = peerServer.Start(); grpcErr != nil { // 监听端口（7051）提供服务
			grpcErr = fmt.Errorf("grpc server exited with error: %s", grpcErr)
		} else {
			logger.Info("peer server exited")
		}
		serve <- grpcErr // 若因发生错误而退出，则发送错误到serve通道
	}()
		// 向进程文件中写入运行进程ID
	if err := writePid(config.GetPath("peer.fileSystemPath")+"/peer.pid", os.Getpid()); err != nil {
		return err
	}

	// Start the event hub server
	// 启动基于专用事件监听端口的gRPC服务器（7053端口，已注册EventHub事件服务器）
	if ehubGrpcServer != nil {
		go ehubGrpcServer.Start()
	}

	// Start profiling http endpoint if enabled
	// 如果打开profile使能标志位，则启动提供服务
	if viper.GetBool("peer.profile.enabled") {
		go func() { // 启动go profile服务器，如果出错，则不会发送错误信息，只是记录到日志里
			// 获取profile监听地址
			profileListenAddress := viper.GetString("peer.profile.listenAddress")
			logger.Infof("Starting profiling server with listenAddress = %s", profileListenAddress)
			if profileErr := http.ListenAndServe(profileListenAddress, nil); profileErr != nil {
				logger.Errorf("Error starting profiler: %s", profileErr)
			}
		}()
	}

```

至此，Peer节点及其功能服务器启动完毕.


-----


## 参考 

> https://github.com/blockchainGuide/

