> 死磕hyperledger fabric源码|Endorser节点背书服务
>
> 文章及代码：https://github.com/blockchainGuide/
>
> 分支：v1.1.0

![77613ebad5a9eb5ea42dbb3b323fdaf5](https://tva1.sinaimg.cn/large/008eGmZEgy1gn2cdssqtrj30zk0m8dhn.jpg)

## 背书概述

Endorser背书节点提供`ProcessProposal()`服务接口用于接收与处理签名提案消息的请求，启动用户链码容器，执行调用链码，并对模拟执行结果进行签名背书，。`Peer`节点启动时解析`core.yaml`文件中的`peer.handlers`配置项，并构造认证过滤器列表。如果存在合法类型的认证过滤器，则需要先经过所有认证过滤器调用`ProcessProposal`()方法进行验证过滤，例如检查身份证书是否过期，然后再提交给背书服务器的`serverEndorser.ProcessProposal`()方法进行处理。 方法功能如下：

```go
func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error) {
	...
	//检查并检验签名提案消息的合法性
	vr, err := e.preProcess(signedProp)
	...
	// 创建交易模拟器与历史查询执行器
	var txsim ledger.TxSimulator
	var historyQueryExecutor ledger.HistoryQueryExecutor
	if chainID != "" {
		// 创建交易模拟器对象
		if txsim, err = e.s.GetTxSimulator(chainID, txid); err != nil {
			return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
		}
		if historyQueryExecutor, err = e.s.GetHistoryQueryExecutor(chainID); err != nil {
			// 创建历史查询器对象
			return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
		}
		// 将历史查询执行器添加到context中的KV键值对
		ctx = context.WithValue(ctx, chaincode.HistoryQueryExecutorKey, historyQueryExecutor)
	}
	
	// 模拟交易执行
	cd, res, simulationResult, ccevent, err := e.simulateProposal(ctx, chainID, txid, signedProp, prop, hdrExt.ChaincodeId, txsim)
	if err != nil {
		// 检查交易模拟运行结果的响应消息
		return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
	}
	if res != nil {
		...
			// 创建背书失败的提案响应消息
			pResp, err := putils.CreateProposalResponseFailure(prop.Header, prop.Payload, res, simulationResult, cceventBytes, hdrExt.ChaincodeId, hdrExt.PayloadVisibility)
			if err != nil {
				return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
			}

			return pResp, &chaincodeError{res.Status, res.Message}
		}
	}

	// 调用ESCC系统链码对模拟执行结果进行背书，并回复提案响应消息
	var pResp *pb.ProposalResponse
	if chainID == "" {
		pResp = &pb.ProposalResponse{Response: res}
	} else { // 签名背书
		pResp, err = e.endorseProposal(ctx, chainID, txid, signedProp, prop, res, simulationResult, ccevent, hdrExt.PayloadVisibility, hdrExt.ChaincodeId, txsim, cd)
		if err != nil {
			return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
		}
		if pResp != nil {
			if res.Status >= shim.ERRORTHRESHOLD { // 检查响应消息是否存在错误
				endorserLogger.Debugf("[%s][%s] endorseProposal() resulted in chaincode %s error for txid: %s", chainID, shorttxid(txid), hdrExt.ChaincodeId, txid)
				return pResp, &chaincodeError{res.Status, res.Message}
			}
		}
	}
	pResp.Response.Payload = res.Payload // 设置链码提案响应消息负载字节数组,含有链码调用返回值
```

主要做了以下几件事：

1. 调用`preProcess()`方法预处理签名提案消息，验证消息合法性
2. 调用`simulateProposal()`方法启动链码容器并模拟执行提案，将结果读写集记录到模拟交易器中。
3. 调用`endorseProposal()`方法对模拟执行结果进行签名背书，并返回提案响应消息。 

下面的内容将会紧紧围绕这几部分来进行分析。

## 预处理签名提案消息

进入到`preProcess`函数：

①： *验证签名提案消息格式与签名的合法性*

```go
prop, hdr, hdrExt, err := validation.ValidateProposalMessage(signedProp)
```

②:  *检查提案消息是否允许外部调用的系统链码*

```go
//解析消息通道头部ChannelHeader结构
	chdr, err := putils.UnmarshalChannelHeader(hdr.ChannelHeader)
	...
	//解析消息签名头部SignatureHeader结构
	shdr, err := putils.GetSignatureHeader(hdr.SignatureHeader)
	...
	//如果是系统链码，则检查是否为允许从外部调用的系统链码：cscc、lscc或qscc
	if e.s.IsSysCCAndNotInvokableExternal(hdrExt.ChaincodeId.Name) {
		endorserLogger.Errorf("Error: an attempt was made by %#v to invoke system chaincode %s",
			shdr.Creator, hdrExt.ChaincodeId.Name)
		err = errors.Errorf("chaincode %s cannot be invoked through a proposal", hdrExt.ChaincodeId.Name)
		//构造提案响应消息对象：状态码为500（错误）与错误信息
		vr.resp = &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}
		return vr, err
	}
```

③：*检查签名提案消息的唯一性以及是否满足指定通道的访问权限策略*

```go
chainID := chdr.ChannelId //获取通道标识号ChannelID，即链chainID
	//// 检查账本中交易ID的唯一性。注意ValidateProposalMessage()方法已经验证了交易号ID的合法性
	txid := chdr.TxId
	if txid == "" {
		err = errors.New("invalid txID. It must be different from the empty string")
		vr.resp = &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}
		return vr, err
	}
	endorserLogger.Debugf("[%s][%s] processing txid: %s", chainID, shorttxid(txid), txid)
	if chainID != "" {
		// 根据交易ID从账本中获取指定的交易对象，检查账本中交易对象的唯一性，
		// 若找到该对象则说明重复发起了交易，此时应报错
		if _, err = e.s.GetTransactionByID(chainID, txid); err == nil {
			return vr, errors.Errorf("duplicate transaction found [%s]. Creator [%x]", txid, shdr.Creator)
		}
		/// 检查是否为系统链码，确保是用户链码
		if !e.s.IsSysCC(hdrExt.ChaincodeId.Name) {
			//// 检查提案是否符合WRITER写通道权限策略
			if err = e.s.CheckACL(signedProp, chdr, shdr, hdrExt); err != nil {
				vr.resp = &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}
				return vr, err
			}
		}
  } else {}
```

以上3个部分内容还需要进一步的细化，看接下来的分析。

### 验证消息格式与签名合法性 

①：调用`validation.ValidateProposalMessage()`函数，以检查签名提案消息格式与签名的合法性，解析获取提案消息、消息头部及其扩展域。

1.1  校验header

```go
chdr, shdr, err := validateCommonHeader(hdr)
```

校验header里面大概做了这几件事：

- `validateChannelHeader(chdr`)函数检查通道头部`chdr`的合法性，其通道头部类型应该属于`ENDORSER_TRANSACTION`、`CONFIG_UPDATE`、`CONFIG`或`PEER_RESOURCE_UPDATE`，并且`Epoch`字段应该为0； 

- `validateSignatureHeader(shdr)`函数检查签名头部`shdr`的合法性，随机数`Nonce`和消息 

  签名者`Creator`不应该为`nil`，并且该对象字节数不为0

1.2 检查消息签名的合法性

```go
err = checkSignatureFromCreator(shdr.Creator, signedProp.Signature, signedProp.ProposalBytes, chdr.ChannelId)
```

该方法先获取当前通道的身份反序列化组件`mspObj`，解析出该签名头部的签名者`creator`，并调用`creator.Validate()`方法，验证`creator`是否为MSP有效的X.509合法证书。然后，调用`creator.Verify()`方法获取哈希方法及消息摘要（哈希值），通过所属`MSP`组件的`BCCSP`加密安全组件调用`id.msp.bccsp.Verify()`方法，验证消息签名的真实性

1.3 验证提案消息头部中的交易ID是否计算正确

```go
err = utils.CheckProposalTxID(
		chdr.TxId,
		shdr.Nonce,
		shdr.Creator)
```

重新计算消息随机数`Nonce`（防止重放攻击）与签名者`Creator`组合信息后的哈希值，并且与交易ID进行比较。如果两者匹配相同，则说明交易ID是正确的。

### 检查是否为允许外部调用的系统链码

```go
if e.s.IsSysCCAndNotInvokableExternal(hdrExt.ChaincodeId.Name) {
		endorserLogger.Errorf("Error: an attempt was made by %#v to invoke system chaincode %s",
			shdr.Creator, hdrExt.ChaincodeId.Name)
		err = errors.Errorf("chaincode %s cannot be invoked through a proposal", hdrExt.ChaincodeId.Name)
		//构造提案响应消息对象：状态码为500（错误）与错误信息
		vr.resp = &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}
		return vr, err
	}
```

### 检查签名提案消息的唯一性 

preProcess()方法继续检查签名提案消息的唯一性，以防止重放攻击。该方法从提案消息通道头部提取链与交易ID，包括两种情况:

- 如果链ID不是空字符串，则需要检查该交易ID的唯一性，确保之前没有提交过该交易到账本中。即根据交易ID从账本的区块文件以及区块索引数据库获取交易数据与交易验证码，并构造成已处理的交易对象 。如果获取交易数据成功且没有错误，则说明账本中已经保存了指定交易ID的交易数据。因此，当前提案消息属于重复提交，报错返回。否则，就说明该签名提案消息通过了消息唯一性的检查；
- 如果链ID是空字符串，则不需要检查签名提案消息的唯一性与验证通道访问权限策略，只需要通过`ValidateProposalMessage()`函数验证该提案消息的合法性即可。

### 检查是否满足通道的访问权限策略

首先调用`IsSysCC`函数，检查链码是否为系统链码。如果是用户链码，则调用 `CheckACL`方法，检查签名提案消息是否满足通道`PROPOSE`权限策略要求，以允许提交该消息到指定通道上继续进行处理。`CheckACL`方法如下：

```go
func (d *defaultACLProvider) CheckACL(resName string, channelID string, idinfo interface{}) error {
  policy := d.defaultPolicy(resName, true)
  ....
  case *pb.SignedProposal:
		return d.policyChecker.CheckPolicy(channelID, policy, idinfo.(*pb.SignedProposal))
	case *common.Envelope:
		sd, err := idinfo.(*common.Envelope).AsSignedData()
		if err != nil {
			return err
		}
		return d.policyChecker.CheckPolicyBySignedData(channelID, policy, sd)
}
```

方法先调用`defaultPolicy()`方法，从全局通道资源策略字典`cResourcePolicyMap`中获取指定策略名称resources.PROPOSE的默认策略。对于`SignedProposal`类型的签名提案消息，`CheckACL()`方法调用`d.policyChecker.CheckPolicy()`方法，检查该签名提案消息是否满足该通道上的`Writers`写权限策略要求。

## 模拟执行提案

ProcessProposal()方法启动**链码容器**与**初始化链码执行环境**，模拟执行合法的签名提案消息，并将模拟执行结果记录在交易模拟器中。其中，对公有数据（包含公共数据与隐私数据哈希值）继续签名背书，并提交给`Orderer`节点请求排序出块，同时将隐私数据通过`Gossip`消息协议发送到组织内的其他授权节点上。核心函数如下：

```go
func (e *Endorser) simulateProposal(ctx context.Context, chainID string, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal, cid *pb.ChaincodeID, txsim ledger.TxSimulator) (resourcesconfig.ChaincodeDefinition, *pb.Response, []byte, *pb.ChaincodeEvent, error) {
	// 解析获取链码调用规范对象
	cis, err := putils.GetChaincodeInvocationSpec(prop)
	...
	//1 检查是否为系统链码
	if !e.s.IsSysCC(cid.Name) { // 如果是调用用户链码，则需要保证该链码已经实例化了
		// === 用户链码，通过调用LSCC系统链码获取账本中保存的链码数据对象ChaincodeData结构
		// 如果链上有链码数据对象，则说明链码已经成功实例化
		cdLedger, err = e.s.GetChaincodeDefinition(ctx, chainID, txid, signedProp, prop, cid.Name, txsim)
		if err != nil {
			return nil, nil, nil, nil, errors.WithMessage(err, fmt.Sprintf("make sure the chaincode %s has been successfully instantiated and try again", cid.Name))
		}
		// 获取已保存的链码版本
		version = cdLedger.CCVersion()
		// 检查提案中的实例化策略与调用账本中的实例化策略是否匹配
		err = e.s.CheckInstantiationPolicy(cid.Name, version, cdLedger)
		if err != nil {
			return nil, nil, nil, nil, err
		}
	} else { // === 执行系统链码，如lscc等
		version = util.GetSysCCVersion() // 获取系统链码版本
	}
	...
	// 2 启动链码容器调用链码
	res, ccevent, err = e.callChaincode(ctx, chainID, version, txid, signedProp, prop, cis, cid, txsim)
	if err != nil {
		endorserLogger.Errorf("[%s][%s] failed to invoke chaincode %s, error: %+v", chainID, shorttxid(txid), cid, err)
		return nil, nil, nil, nil, err
	}
	//3 获取并处理交易模拟执行结果
	if txsim != nil {
		if simResult, err = txsim.GetTxSimulationResults(); err != nil {
			return nil, nil, nil, nil, err
		}

		if simResult.PvtSimulationResults != nil { // 检查模拟结果隐私数据的合法性
			if cid.Name == "lscc" {
				// TODO: remove once we can store collection configuration outside of LSCC
				// 分发隐私数据
				return nil, nil, nil, nil, errors.New("Private data is forbidden to be used in instantiate")
			}
			if err := e.distributePrivateData(chainID, txid, simResult.PvtSimulationResults); err != nil {
				return nil, nil, nil, nil, err
			}
		}
		// 分发隐私数据
		if pubSimResBytes, err = simResult.GetPubSimulationBytes(); err != nil {
			return nil, nil, nil, nil, err
		}
	}
	return cdLedger, res, pubSimResBytes, ccevent, nil
}
```



### 根据链码类型执行不同实例化策略

首先调用`GetChaincodeInvocationSpec`函数，从提案消息中解析提取出链码调用规范对象，然后调用`IsSysCC(cid.Name)`方法，依次匹配默认的系统链码名称，以判断当前链码类型是用户链码还是系统链码，分为用户链码和系统链码两种情况检查实例化策略。

①：*用户链码*

```go
if !e.s.IsSysCC(cid.Name) { // 如果是调用用户链码，则需要保证该链码已经实例化了
		// === 用户链码，通过调用LSCC系统链码获取账本中保存的链码数据对象ChaincodeData结构
		// 如果链上有链码数据对象，则说明链码已经成功实例化
		cdLedger, err = e.s.GetChaincodeDefinition(ctx, chainID, txid, signedProp, prop, cid.Name, txsim)
		if err != nil {
			return nil, nil, nil, nil, errors.WithMessage(err, fmt.Sprintf("make sure the chaincode %s has been successfully instantiated and try again", cid.Name))
		}
		// 获取已保存的链码版本
		version = cdLedger.CCVersion()
		// 检查提案中的实例化策略与调用账本中的实例化策略是否匹配
		err = e.s.CheckInstantiationPolicy(cid.Name, version, cdLedger)
		if err != nil {
			return nil, nil, nil, nil, err
		}
```

②：系统链码

```go
else { // === 执行系统链码，如lscc等
		version = util.GetSysCCVersion() // 获取系统链码版本
	}
```



### 启动链码容器

```go
res, ccevent, err = e.callChaincode(ctx, chainID, version, txid, signedProp, prop, cis, cid, txsim)
```

①：*设置context上下文对象中交易模拟器的KV键值对，其中，键为TXSimulatorKey，值为交易模拟器txsim*

```go
if txsim != nil {
		ctxt = context.WithValue(ctxt, chaincode.TXSimulatorKey, txsim)
	}
```

②：*根据链码名称检查是否为系统链码*

```go
scc := e.s.IsSysCC(cid.Name)
```

③：*执行链码调用*

```go
res, ccevent, err = e.s.Execute(ctxt, chainID, cid.Name, version, txid, scc, signedProp, prop, cis)
```

④：*检查调用链码名称lscc*

```go
// 第1个参数为deploy部署或upgrade升级，第2个参数是链ID，第3个是链码部署规范对象
	if cid.Name == "lscc" && len(cis.ChaincodeSpec.Input.Args) >= 3 && (string(cis.ChaincodeSpec.Input.Args[0]) == "deploy" || string(cis.ChaincodeSpec.Input.Args[0]) == "upgrade") {
		var cds *pb.ChaincodeDeploymentSpec
		// 获取并验证链码部署规范
		cds, err = putils.GetChaincodeDeploymentSpec(cis.ChaincodeSpec.Input.Args[2])
		if err != nil {
			return nil, nil, err
		}

		//this should not be a system chaincode
		// 若试图部署/升级系统链码，则报错
		if e.s.IsSysCC(cds.ChaincodeSpec.ChaincodeId.Name) {
			return nil, nil, errors.Errorf("attempting to deploy a system chaincode %s/%s", cds.ChaincodeSpec.ChaincodeId.Name, chainID)
		}
		// 执行部署/升级链码
		_, _, err = e.s.Execute(ctxt, chainID, cds.ChaincodeSpec.ChaincodeId.Name, cds.ChaincodeSpec.ChaincodeId.Version, txid, false, signedProp, prop, cds)
		if err != nil {
			return nil, nil, err
		}
```

启动的真正过程正是在`e.s.Execute`中完成的,分析如下：

> /core/chaincode/exectransaction.go/Execute()

```go
func Execute(ctxt context.Context, cccid *ccprovider.CCContext, spec interface{}) (*pb.Response, *pb.ChaincodeEvent, error) {
	...
	// === 设置初始链码消息对象
	// 部署（实例化）deploy命令或升级upgrade命令：调用链码Init()接口方法
	cctyp := pb.ChaincodeMessage_INIT
	//// 检查链码规范对象类型为ChaincodeDeploymentSpec或ChaincodeInvocationSpec
	if cds, _ = spec.(*pb.ChaincodeDeploymentSpec); cds == nil {
		if ci, _ = spec.(*pb.ChaincodeInvocationSpec); ci == nil {
			panic("Execute should be called with deployment or invocation spec")
		}
		// 调用invoke或查询query命令等：调用链码Invoke()接口方法
		cctyp = pb.ChaincodeMessage_TRANSACTION
	}

	// === 启动链码容器，返回链码输入参数等
	// created->established->ready状态
	_, cMsg, err := theChaincodeSupport.Launch(ctxt, cccid, spec)
	...
	// === 模拟执行交易链码并等待完成，监听并返回resp响应结果消息
	resp, err := theChaincodeSupport.Execute(ctxt, cccid, ccMsg, theChaincodeSupport.executetimeout)
	...
	// === 处理模拟执行结果
	if resp.ChaincodeEvent != nil {
		....
	}
		....
}
```

启动的动作在下面这个方法中完成：

```go
, cMsg, err := theChaincodeSupport.Launch(ctxt, cccid, spec)
```

此方法的核心又是：`launchAndWaitForRegister()`，负责具体的链码容器工作,代码位置：**/core/chaincode/chaincode_support.go/launchAndWaitForRegister**

```go
func (chaincodeSupport *ChaincodeSupport) launchAndWaitForRegister(ctxt context.Context, cccid *ccprovider.CCContext, cds *pb.ChaincodeDeploymentSpec, launcher launcherIntf) error {
	...
	// 如果chaincodeMap字典中已经存在对应的链码规范名称，则说明已经启动链码容器，此时直接返回即可
	if _, hasBeenLaunched := chaincodeSupport.chaincodeHasBeenLaunched(canName); hasBeenLaunched {
		...
	}
	// 检查该链码容器是否已经正常运行，直接返回
	if chaincodeSupport.launchStarted(canName) {
		...
	}
	...
		// 核心方法：启动容器，实际调用的是ccLauncherImpl方法
		resp, err := launcher.launch(ctxt, notfy)
	...
	// === 阻塞等待处理响应消息，等待REGISTER链码消息
	select {
	case ok := <-notfy:
		// Peer侧接收到链码容器侧发来的REGISTER注册链码消息，触发Handler的FSM运行，
		// 在回调方法beforeregister()中将外层Handler传递的notfy通道注册到Peer侧Handler中，
		// 根据链码注册成功结果，将结果消息放入notfy通道，触发此处的select语句。
		// 若notfy为flase，则说明注册失败。反之，则说明注册成功
		...
}
```

至此，`chaincode.Execute()`函数检查并启动了链码容器，执行完成链码请求操作.

### 处理模拟执行结果

处理模拟执行结果是由下面几段代码实现的：

```go
//=== 获取并处理交易模拟执行结果
	if txsim != nil {
		if simResult, err = txsim.GetTxSimulationResults(); err != nil {
			return nil, nil, nil, nil, err
		}

		if simResult.PvtSimulationResults != nil { // 检查模拟结果隐私数据的合法性
			if cid.Name == "lscc" {
				// TODO: remove once we can store collection configuration outside of LSCC
				return nil, nil, nil, nil, errors.New("Private data is forbidden to be used in instantiate")
			}
			// 分发隐私数据
			if err := e.distributePrivateData(chainID, txid, simResult.PvtSimulationResults); err != nil {
				return nil, nil, nil, nil, err
			}
		}

		if pubSimResBytes, err = simResult.GetPubSimulationBytes(); err != nil {
			return nil, nil, nil, nil, err
		}
	}
```

两个关键函数：一个是`GetTxSimulationResults`，还有一个就是`distributePrivateData`。

`GetTxSimulationResults`主要获取交易模拟执行结果的隐私数据读写集，然后遍历计算集合隐私数据的哈希值，然后获取交易模拟执行结果的公有数据读写集，最后构造交易模拟执行结果`TxSimulationResults`结构对象并返回。

 `distributePrivateData`首先会*获取指定通道上的隐私数据处理句柄*，然后通过`handler.distributor.Distribute`分发隐私数据，

最后通过`coordinator`模块将指定交易`txID`的隐私数据读写集`privData`暂时保存到本地`transient`隐私数据库中。`Committer`记账节点在提交区块数据与隐私数据之后，主动删除`transient`隐私数据库中关联的隐私数据，以及时清理过期数据。



## 对模拟执行结果签名背书

`endorseProposal()`方法对模拟执行结果进行签名背书，并返回提案响应消息。

```go
func (e *Endorser) endorseProposal(...) (*pb.ProposalResponse, error) {
	...
	// 调用ESCC系统链码进行背书
	res, _, err := e.callChaincode(ctx, chainID, version, txid, signedProp, proposal, ecccis, &pb.ChaincodeID{Name: escc}, txsim)
	...
}
```

`callChaincode()`方法调用`ESCC`系统链码的`EndorserOneValidSignature.Invoke`()方法，对模拟结果执行签名背书操作。代码如下：

位置：`/core/scc/escc/endorser_onevalidsignature.go/Invoke`

```go
func (e *EndorserOneValidSignature) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
	args := stub.GetArgs() // 获取参数列表
	// 检测参数个数
	if len(args) < 6 {
		return shim.Error(fmt.Sprintf("Incorrect number of arguments (expected a minimum of 5, provided %d)", len(args)))
	} else if len(args) > 8 {
		return shim.Error(fmt.Sprintf("Incorrect number of arguments (expected a maximum of 7, provided %d)", len(args)))
	}
	...
	// 获取执行链码响应消息
	response, err := putils.GetResponse(args[4])
	...
	// 获取模拟执行结果
	results = args[5]

	/..
	// 获取本地MSP组件
	localMsp := mspmgmt.GetLocalMSP()
	...
	// 获取本地默认签名者身份实体（即背书成员）
	signingEndorser, err := localMsp.GetDefaultSigningIdentity()
	...
	// 创建签名的提案响应消息
	presp, err := utils.CreateProposalResponse(hdr, payl, response, results, events, ccid, visibility, signingEndorser)
	...
	// 序列化提案响应字节数组
	prBytes, err := utils.GetBytesProposalResponse(presp)
	...
	// 回复执行成功消息
	return shim.Success(prBytes)
}

```

至此，`Endorser`背书节点处理签名提案消息的流程结束。

## 参考 

> https://github.com/blockchainGuide/

