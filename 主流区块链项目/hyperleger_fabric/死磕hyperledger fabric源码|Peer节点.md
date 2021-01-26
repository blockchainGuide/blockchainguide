> 死磕hyperledger fabric源码|Peer节点
>
> 文章及代码：https://github.com/blockchainGuide/
>
> 分支：v1.0.0



## 概述

peer节点主要为fabric处理交易和存储区块。



## 目录结构

```go
|-
```





## Endorser

>  主体A接收到peer B发送的申请消息，A签名检查通过，然后告诉B认同此申请，从而此消息才可能被提交，A的角色就是Endorser(背书者)
>
> 注：详细的背书策略不仅仅是简单的签名检查通过这个条件。



### 背书的实现框架及细节

**服务的原型： protos/peer/peer.proto**

```go
service Endorser {
	rpc ProcessProposal(SignedProposal) returns (ProposalResponse) {}
}
```

**服务接口和注册函数：protos/peer/peer.pb.go**

```go
type EndorserClient interface {
	ProcessProposal(ctx context.Context, in *SignedProposal, opts ...grpc.CallOption) (*ProposalResponse, error)
}
```

```go
func RegisterEndorserServer(s *grpc.Server, srv EndorserServer) {
	s.RegisterService(&_Endorser_serviceDesc, srv)
}
```

**Endorser服务的实现：core/endorser/endorser.go**

```go
type Endorser struct {
	policyChecker policy.PolicyChecker
}
```

可以看出`Endorser`的核心实现就是`ProcessProposal`，来处理客户端发送的`SignedProposal`数据，返回`ProposalResponse`数据。

我们来查看`ProcessProposal`函数,主要做了以下几件事：

①：检查`signnedproposal`消息的有效性,返回`Header`和`ChaincodeHeaderExtension`消息

```go
func ValidateProposalMessage(signedProp *pb.SignedProposal) (*pb.Proposal, *common.Header, *pb.ChaincodeHeaderExtension, error) {
  ...
  // 从signedProp中提取prorosal 消息
	prop, err := utils.GetProposal(signedProp.ProposalBytes)
  
  ...
  // 查看proposal的header
	hdr, err := utils.GetHeader(prop.Header)
  
  // 校验header
	chdr, shdr, err := validateCommonHeader(hdr)
  
  // 校验签名
	err = checkSignatureFromCreator(shdr.Creator, signedProp.Signature, signedProp.ProposalBytes, chdr.ChannelId)
  
  //检查交易ID是否被正确计算以及确保找到重复的交易
  err = utils.CheckProposalTxID(chdr.TxId,shdr.Nonce,shdr.Creator)
  
  ...
  case common.HeaderType_ENDORSER_TRANSACTION:
  //  校验proposal的消息类型为chaincode
		chaincodeHdrExt, err := validateChaincodeProposalMessage(prop, hdr)
}
```

1.1 `validateCommonHeader`做了以下事情：

- `validateChannelHeader`：校验`header type`必须为：**HeaderType_ENDORSER_TRANSACTION**、**HeaderType_CONFIG_UPDATE**、**HeaderType_CONFIG**三者之一。
- `validateSignatureHeader`：校验`Nonce`、`Creator`不为空

1.2 `checkSignatureFromCreator`做了以下事情：

- `creator.Validate()` ：确保**creator**的证书有效
- `creator.Verify(msg, sig)` ：**creator** 签名校验要通过

1.3 `CheckProposalTxID`做了以下事情：

- `ComputeProposalTxID`：检查**txid**是否等于**nonce**和**creator**连接后得到的哈希值。

1.4 `validateChaincodeProposalMessage`做了以下事情：

- `chaincodeHdrExt.PayloadVisibility != nil`：验证了**Proposal** 中**Extension**对应的结构体中的**PayloadVisibility**不为空。

②：根据返回的**Header**获取*ChannelHeader*和*SignatureHeader*

```go
chdr, err := putils.UnmarshalChannelHeader(hdr.ChannelHeader)
...
shdr, err := putils.GetSignatureHeader(hdr.SignatureHeader)
...
```

③：验证正在处理的*SignedProposal*涉及的*chaincode*的 ID 是否是系统的，要确保可以被外部调用，通过字段*InvokeableExternal*来查看。

```go
if syscc.IsSysCCAndNotInvokableExternal(hdrExt.ChaincodeId.Name){}
```

④：获取*peer*本地的账本，根据交易ID从账本中查询记录

```go
lgr := peer.GetLedger(chainID)
lgr.GetTransactionByID(txid);
```

⑤：判断是否为系统链码，不是的话，要检测*ACL*的访问权限

```go
if !syscc.IsSysCC(hdrExt.ChaincodeId.Name) {
			// check that the proposal complies with the channel's writers
			if err = e.checkACL(signedProp, chdr, shdr, hdrExt); err != nil {
				return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
			}
		}
```

⑥：当*chainId*不为空的时候，获取交易模拟对象和账本历史查询对象模拟交易

```go
if txsim, err = e.getTxSimulator(chainID); err != nil {
			...
		}
		if historyQueryExecutor, err = e.getHistoryQueryExecutor(chainID); err != nil {
			...
		}
defer txsim.Done()
}
```

⑦：通过调用*chaincode*模拟*Proposal*

```go
//1. 校验ESCC和VSCC
if err = e.checkEsccAndVscc(prop); err != nil {
		...
	}

//从LSCC中获取指定名字的ChainCode的数据
e.getCDSFromLSCC(ctx, chainID, txid, signedProp, prop, cid.Name, txsim)

//执行ChainCode事件，并返回HTTP状态以及Chaincode事件
res, ccevent, err = e.callChaincode(ctx, chainID, version, txid, signedProp, prop, cis, cid, txsim)

//返回ChainCode数据
txsim.GetTxSimulationResults();
```

⑧：背书者对模拟交易进行背书

最终获取到交易申请应答数据，最终也是调用的*CallChainCode*,背书功能会使用系统*ChainCode*中用于背书的*escc*.

```go
pResp, err = e.endorseProposal(ctx, chainID, txid, signedProp, prop, res, simulationResult, ccevent, hdrExt.PayloadVisibility, hdrExt.ChaincodeId, txsim, cd)
```



## committer记账节点

记账节点负责验证交易和提交账本，包括公有数据（即区块数据，包括公共数据和私密数据hash值）与私密数据。在提交账本前需要验证交易数据的有效性，包括交易消息的格式、签名有效性以及调用VSCC验证消息的合法性及指定背书策略的有效性，接着通过MVCC检查读写集冲突并标记交易的有效性，最后提交区块数据到区块文件系统，建立索引信息并保存到区块索引数据库，更新有效交易和私密数据到状态数据库，将经过背书节点到有效交易同步到历史数据库，并更新隐私数据库。

主要就以下几个文件：

>|-core/committer/txvalidator/validator.go
>
>|-core/committer/committer.go
>
>|-core/committer/committer_impl.go

### committer.go

主要声明了以下接口：

```go
//提交块到账本
Commit(block *common.Block) error

//返回最近的块序列号
LedgerHeight() (uint64, error)

//获取带有切片中提供的序列号的块
GetBlocks(blockSeqs []uint64) []*common.Block

//关闭commit服务
Close()

```

### committer_impl.go

#### 提交块到账本

```go
func (lc *LedgerCommitter) Commit(block *common.Block) error {
  // 校验并标记有效交易
  if err := lc.validator.Validate(block); err != nil {
		return err
	}
  
  //提交块
  if err := lc.ledger.Commit(block); err != nil {
		return err
	}
  
  //发送块事件
  if err := producer.SendProducerBlockEvent(block); err != nil {
		logger.Errorf("Error publishing block %d, because: %v", block.Header.Number, err)
	}
}
```

主要也就是做了三件事：

1. 校验并标记有效交易
2. 提交块
3. 发送块事件

#### 返回最近的块序列号

```go
func (lc *LedgerCommitter) LedgerHeight() (uint64, error) {
  ...
  if info, err = lc.ledger.GetBlockchainInfo(); err != nil {
    ....
  }
}
```

#### 获取块

```go
func (lc *LedgerCommitter) GetBlocks(blockSeqs []uint64) []*common.Block {
  if blck, err := lc.ledger.GetBlockByNumber(seqNum); err != nil {
    ....
  }
}
```

### validator.go

分析如下：

#### 声明支持评估VSCC的接口

```go
type Support interface {
	// 返回和此validator关联的账本
	Ledger() ledger.PeerLedger

	// 返回此链的MSP管理器
	MSPManager() msp.MSPManager

	// 尝试应用一个configtx来成为新的配置
	Apply(configtx *common.ConfigEnvelope) error

	//返回在通道中定义的应用程序MSPs的id
	GetMSPIDs(cid string) []string
}
```

#### 声明Validator接口

> 验证器接口，定义了验证块事务的API，返回无效事务的位数组掩码。

```go
type Validator interface {
	Validate(block *common.Block) error
}
```

#### 声明VSCC接口

> 私有接口来解耦tx验证器和vscc的执行，以增加可测试性的txValidator
>

```go
type vsccValidator interface {
	VSCCValidateTx(payload *common.Payload, envBytes []byte, env *common.Envelope) (error, peer.TxValidationCode)
}
```

#### 校验交易

```go
func (v *txValidator) Validate(block *common.Block) error {
  ...
  //校验交易索引
  if payload, txResult = validation.ValidateTransaction(env); txResult != peer.TxValidationCode_VALID {}
  
  ...
  // 解码通道的header
  chdr, err := utils.UnmarshalChannelHeader(payload.Header.ChannelHeader)
  
  ...
  // 删除不在链上的交易
  if !v.chainExists(channel) {...}
  
  ...
  
  
  if common.HeaderType(chdr.Type) == common.HeaderType_ENDORSER_TRANSACTION {
    //校验交易是否重复
     if _, err := v.support.Ledger().GetTransactionByID(txID); err == nil {}
    
    // VSCC方式验证交易
    err, cde := v.vscc.VSCCValidateTx(payload, d, env)
    
    //从交易上获取chaincode实例
    invokeCC, upgradeCC, err := v.getTxCCInstance(payload)
  }else if common.HeaderType(chdr.Type) == common.HeaderType_CONFIG{
    //解码出config
    configEnvelope, err := configtx.UnmarshalConfigEnvelope(payload.Data)
    
    //校验config
    if err := v.support.Apply(configEnvelope); err != nil {}
    
  }else{
    //未知的交易类型
  }
  
}
```

peer节点相关的也就这么几部分内容。



## 参考 

> https://github.com/blockchainGuide/

