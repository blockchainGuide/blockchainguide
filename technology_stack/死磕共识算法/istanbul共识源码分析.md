## handler.go （处理消息）

### start 

1. startNewRound(common.Big0)
2. subscribeEvents()
3. handleEvents()
   - istanbul.RequestEvent{}
   - istanbul.MessageEvent{}
   - backlogEvent{}
   - timeoutEvent{}
   - istanbul.FinalCommittedEvent{}

### stop

1. roundChangeTimer.Stop()
2. unsubscribeEvents()
   - events.Unsubscribe()timeoutSub.Unsubscribe()
   - finalCommittedSub.Unsubscribe()
3. 使得handler处于wait状态

### handleMsg&handleTimeoutMsg



1. 先进行msg的check
2. 如果是futureMsg，直接存储并returnErr 
3. 接下来处理4个msg 


## startNewRound

1. 首先设置roundChange 为false，如果最新的proposer和proposal不存在，直接return

2. 第二个步骤有几个if else 需要拆解（TODO）

3. 如果roundChange为true，新建一个View，如果为false，sequence加1 

4. 选出proposer（CalcProposer），根据valset、lastprotser、round进行选择

5. 设置状态为接收请求c.setState(StateAcceptRequest)。

   - 发送istanbul.RequestEvent（把proposal扔出去）
   - 处理积压消息（Preprepare）
   - 发送backlogEvent事件
   - endPreprepare：

   - 如果自己是proposer并且和proposal有着相同的sequence，那就广播preprepare消息
   - 广播途中将消息转换成payload 并返回

   

## handlePreprepare 

   

   - 校验message ，如果是老的prepare消息，要commit 这个proposal,直接会到广播comiit消息
   - 校验消息来自当前的proposer
   - 校验我们接收到的proposal
     - check 坏块
     - check block body(rehash)
     - verifyHeader (TODO,重点了解)
   - 校验之后，（TODO，逻辑？）

6. 如果锁定的proposal和接收到的proposal不一致就sendNextRoundChange，如果一样，就接收prepare消息并设置状态为StatePreprepared并且发送commit消息



## handleCommit

1. checkMessage 
2. verifyCommit 
3. acceptCommit(添加到commits中)
4. 有了足够的commit messages并且不是comiited状态将commit proposal 
5. Commit
   - 设置状态为comitted 
   - 创建commitSeals  
   - 进入最终的commit代码
     - 校验proposal是一个有效块
     - seals写入到extra-data中（writeCommittedSeals 关键代码）
     - 更新block的header
     - 如果proposedBlockHash == commitedBlockHash ,那么就把block 扔到commitCh中去seal并且等待seal结果（这是proposer才会走的通道），直接返回；如果不一样，直接塞入到enqueue中（sb.broadcaster.Enqueue(fetcherID, block)）

## handleRoundChange 

1. checkMessage 
2. roundChangeSet 添加消息
3. 只要达到了F+1个roundChange就构成了一个weak proof ,就可以检查此时的round是否比我们的round小，如果小就CatchUp,然后startNewRound（需要2F+1）



## engine.go

### prepare

准备header

1. sb.snapshot

2. 从candidates中随机设置coinbase 

3. prepareExtra（将快照中的validators添加到extraData的validators中），payload 数据就是编码后的IstanbulExtra，作为extra

   > ```go
   > types.IstanbulExtra{
   >    Validators:    vals,
   >    Seal:          []byte{},
   >    CommittedSeal: [][]byte{},
   > }
   > ```



### FinalizeAndAssemble

运行交易后状态更改，组装成最终的block 



### seal

生成一个新块放入给定通道 

1. 主块判断是否是validator，子链还必须判断是不是子validator
2. 更新块（updateBlock），proposer用自己的私钥给块签名生成seal，并写入IstanbulExtra的seal中，包括自己的签名的标记和其他人签名的标记，其实就是返回带签名的块
3. 把块丢到共识引擎中，通过事件istanbul.RequestEvent传播，从而进入到handleRequest，开启sendPrepeare，result等待的就是sb.commitCh中的数据