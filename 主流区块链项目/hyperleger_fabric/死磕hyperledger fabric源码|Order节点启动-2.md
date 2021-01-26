> 死磕hyperledger fabric源码|Order节点启动
>
> 文章及代码：https://github.com/blockchainGuide/
>
> 分支：v1.1.0

![207f698ab0d8c69266fb6fe4c8f85def](https://tva1.sinaimg.cn/large/008eGmZEgy1gmytqt8ef1g30zk0m7dn6.gif)

## Orderer节点启动流程

节点启动开始在`/orderer/common/server/main.go`:

```GO
func Main() {
	fullCmd := kingpin.MustParse(app.Parse(os.Args[1:]))  // 解析用户命令行
	...
	conf, err := config.Load() //  加载orderer.yaml配置文件
	...
	initializeLoggingLevel(conf) //初始化日志级别
	initializeLocalMsp(conf) //初始化本地MSP组件

	prettyPrintStruct(conf) // 打印配置信息
	Start(fullCmd, conf) // 启动Orderer排序服务器
}
```

主要做了以下几件事：
- 解析用户命令行
- 加载orderer.yaml配置文件
- 初始化日志级别
- 初始化本地MSP组件
- 打印配置信息
- 启动Orderer排序服务器

接下来将会展开的去讲上面比较重要的一些内容。

## 加载orderer.yaml配置文件

是由`config.Load()`开启的，进入到`/orderer/common/localconfig/config.go/Load()`

①：初始化viper

调用`InitViper()`函数设置配置文件路径，并默认在`$FABRIC_CFG_PATH`（如`/etc/hyperledger/fabric`）路径下查找配置文件，找不到文件时再依次查找当前目录、默认开发配置目录 （`$GOPATH/src/github.com/hyperledger/fabric/sampleconfig`）和系统默认配置路径 （`/etc/hyperledger/fabric`）。接着开启匹配系统环境变量的模式，即为Viper组件配置项（以.分割的格式）添加指定前缀“ORDERER_”，转换为大写字母形式，再将“.”替换为`_`。这样，Viper组件就能在查找配置项时，与以“ORDERER_”前缀开头的环境变量进行匹配，获取其在环境变量中的配置值。

```go
config := viper.New()
	cf.InitViper(config, configName)
	config.SetEnvPrefix(Prefix)
	config.AutomaticEnv()
	replacer := strings.NewReplacer(".", "_")
	config.SetEnvKeyReplacer(replacer)
```

②：加载Orderer.yaml配置文件

```go
err := config.ReadInConfig()
```

③：解析配置文件成Orderer配置对象

```go
var uconf TopLevel
err = viperutil.EnhancedExactUnmarshal(config, &uconf)
```

此配置对象为TopLevel型，结构体如下：

```go
type TopLevel struct {
	General    General    //通用配置对象
	FileLedger FileLedger //文本账本配置对象
	RAMLedger  RAMLedger  //RAM账本配置对象
	Kafka      Kafka      // Kafka共识组件配置对象
	Debug      Debug      // 调试信息配置对象
}
```

④：检查Orderer配置对象conf配置项并设置默认值

```go
uconf.completeInitialization(filepath.Dir(config.ConfigFileUsed()))
```

## 初始化日志与本地MSP组件

①：初始化日志

`initializeLoggingLevel`设置`Orderer`节点上的日志后端输出流、输出格 式与默认日志级别（`INFO`级别）。

```go
func initializeLoggingLevel(conf *config.TopLevel) {
	flogging.InitBackend(flogging.SetFormat(conf.General.LogFormat), os.Stderr)
	flogging.InitFromSpec(conf.General.LogLevel)
}
```

②：初始化本地MSP组件

首先加载本地的`MSP`组件，根据MSP配置文件路径、`BCCSP`密码服务组件配置、`MSP`名称初始化本地`MSP`组件 。

```go
func initializeLocalMsp(conf *config.TopLevel) {
	err := mspmgmt.LoadLocalMsp(conf.General.LocalMSPDir, conf.General.BCCSP, conf.General.LocalMSPID)
	...
}
```

本地`MSP`组件默认使用`bccspmsp`类型对象。该类型的`MSP`组件是基于`BCCSP`组件提供密码套件服务的，封装了`MSP`组件（通常对应于一个组织）信任的相关证书列表（包含根`CA`证书、中间`CA`证书等）、`MSP`名称、签名者身份实体与管理员身份实体列表等。MSP组件的关键内容后面会有专门去讲解。

## 启动Orderer排序节点

启动函数在`/orderer/common/server/main.go/Start()函数`中，接下来一步步的解析：

①：创建本地MSP签名者实体 signer

```go
signer := localmsp.NewSigner()       
```

②：初始化TLS认证的安全服务器配置项和gRPC服务器

本地gRPC服务器grpcServer默认端口为7050

```go
serverConfig := initializeServerConfig(conf)       
grpcServer := initializeGrpcServer(conf, serverConfig) 
```

③：设置TLS连接认证的回调函数

```go
tlsCallback := func(bundle *channelconfig.Bundle) {
		if grpcServer.MutualTLSRequired() { // 检测是否需要认证TLS客户端证书
			logger.Debug("Executing callback to update root CAs")
			updateTrustedRoots(grpcServer, caSupport, bundle) //执行回调函数更新根CA证书
		}
	}
```

④：创建多通道注册管理器

多通道注册管理器`Registrar`对象，用于注册`Orderer`节点上的所有通道（包括系统通道和应用通道），负责维护通道配置、账本等重要资源。 

多通道注册管理器`Registrar`对象相当于`Orderer`节点上的“资源管理器”，为每个通道创建关联的共识组件链对象，负责交易排序、打包出块、提交账本以及通道管理等工作

```go
manager := initializeMultichannelRegistrar(conf, signer, tlsCallback)
```

```go
func initializeMultichannelRegistrar(conf *config.TopLevel, signer crypto.LocalSigner,
	callbacks ...func(bundle *channelconfig.Bundle)) *multichannel.Registrar {
	lf, _ := createLedgerFactory(conf) //创建通道的账本工厂对象
	if len(lf.ChainIDs()) == 0 {
		initializeBootstrapChannel(conf, lf) //初始化系统通道
	} else {
		logger.Info("Not bootstrapping because of existing chains")
	}

	//创建并设置共识组件字典
	consenters := make(map[string]consensus.Consenter)
	consenters["solo"] = solo.New()
	consenters["kafka"] = kafka.New(conf.Kafka) //// Kafka类型共识组件
	return multichannel.NewRegistrar(lf, consenters, signer, callbacks...)
}
```

4.1 创建通道的账本工厂对象

代码路径`/orderer/common/server/util.go`，大概做了以下几件事：
- 获取`Orderer`节点上的区块账本存储目录ld，包括默认目录`/var/hyperledger/production/orderer`或临时目录中的子目录`hyperledger-fabric-ordererledger`+随机数后缀（默认目录不存在时使用）
- 创建基于文件的区块账本工厂对象lf（`fileLedgerFactory`类型）
- 在区块账本目录下建立以chains命名的子目录（`/var/hyperledger/production/orderer/chains`），由每个通道账本的区块数据存储对象负责在`chains`子目录下创建维护以通道ID（即链ID）命名的通道账本子目录，用于保存该通道账本的所有区块数据文件。其中，区块数据文件名都是以`blockfile_num`命名，`num`是6位区块文件编号，左侧不足位数用0补齐。

4.2 初始化系统通道

```go
func initializeBootstrapChannel(conf *config.TopLevel, lf blockledger.Factory) {
	...
	switch conf.General.GenesisMethod { // 分析创世区块的生成方式
	case "provisional": // 根据配置文件生成创世区块
		genesisBlock = encoder.New(genesisconfig.Load(conf.General.GenesisProfile)).GenesisBlockForChannel(conf.General.SystemChannel)
	case "file": // 根据创世区块文件生成创世区块
		genesisBlock = file.New(conf.General.GenesisFile).GenesisBlock()
	default:
	}

	chainID, err := utils.GetChainIDFromBlock(genesisBlock) // 从创世区块中解析获取通道ID
	...
	gl, err := lf.GetOrCreate(chainID) // 创建系统通道的区块账本对象
	...
	err = gl.Append(genesisBlock) // 添加区块到系统通道账本上
	...
}
```

⑤：创建Orderer排序服务器

`Orderer`排序服务器，提供`Orderer`服务与管理所有通道资源及其账本、共识组件等。

```go
server := NewServer(manager, signer, &conf.Debug, conf.General.Authentication.TimeWindow, mutualTLS)
```

```go
func NewServer(r *multichannel.Registrar, _ crypto.LocalSigner, debug *localconfig.Debug, timeWindow time.Duration, mutualTLS bool) ab.AtomicBroadcastServer {
	s := &server{
		dh:        deliver.NewHandlerImpl(deliverSupport{Registrar: r}, timeWindow, mutualTLS),
		bh:        broadcast.NewHandlerImpl(broadcastSupport{Registrar: r}),
		debug:     debug,
		Registrar: r,
	}
	return s
}
```

- `bh：Broadcast`服务处理句柄（`deliverHandler`类型）。该对象实现了`Broadcast`交易广播服务的`Handle(srv ab.AtomicBroadcast_BroadcastServer)`消息处理接口，负责接收客户端提交的普通交易消息与配置交易消息，并分别进行处理，过滤后转发给通道绑定的共识组件链对象进行处理； 

- `dh：Deliver`服务处理句柄（`handlerImpl`类型）。该对象实现了`Deliver`区块分发服务的`Handle(srv*DeliverServer)`消息处理接口，负责接收客户端提交的区块请求消息，从`Orderer`节点区块账本中读取指定的区块数据，并返回给请求节点。如果请求的指定区块还没有生成，则默认阻塞等待直到该区块创建和提交完毕； 

- `Registrar：Orderer`节点的多通道注册管理器（`Registrar`类型）。该对象封装了`Orderer`节点上所有通道的链支持对象字典`chains`、共识组件字典`consenters`、区块账本工厂对象`ledgerFactory`、系统通道链支持对象与ID、本地签名者实体`signer`等，用于管理通道配置、区块账本对象、共识组件等核心资源，相当于`Orderer`节点上的“资源管理器”。

⑥：解析执行子命令

- start子命令：启动`profile`服务与`Orderer`排序服务器，支持`go tool pprof`命令查看与分析程序性能瓶颈
- benchmark子命令：用于启动测试服务器

```go
switch cmd { //分析命令类型
	case start.FullCommand(): // "start" command // start启动子命令
		logger.Infof("Starting %s", metadata.GetVersionInfo())
		initializeProfilingService(conf) // goroutine启动go profile服务
		ab.RegisterAtomicBroadcastServer(grpcServer.Server(), server)
		logger.Info("Beginning to serve requests")
		grpcServer.Start() // 启动gRPC服务器提供Orderer服务
	case benchmark.FullCommand(): // "benchmark" command // "benchmark" 测试用例子命令
		logger.Info("Starting orderer in benchmark mode")
		benchmarkServer := performance.GetBenchmarkServer()
		benchmarkServer.RegisterService(server)
		benchmarkServer.Start()
	}
```

-----


## 参考 

> https://github.com/blockchainGuide/

