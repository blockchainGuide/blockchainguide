## 运行原理

> https://github.com/b2network/b2-node/issues/93

![image-20240430102213666](https://p.ipic.vip/oorhic.png)



大致的步骤如下：

1. Aggregator 调用 polygonZkevm.sol 的verifyBatchesTrustedAggregator函数（🚩需要确认事件会提供什么数据）

2. B2-committer 监听上述事件保存到MySQL中

3. B2-commit 提交一个proposal 到b2-node （发送交易）

   交易格式如下：

   ```go
   transaction {  
        "proposalId":  getLastProposalId() from b2-node api,
        "stateRoot": rootHash from a merkle tree which contains all the stateRoots of the batch, which are range of startBatchNum and endBatchNum
        "startBatchNum": getFinalBatch() from b2-node rpc or rest interface,
        "endBatchNum": startBatchNum + 10 (defined a constant)
   ```

   实际真正的proposal 生成依赖上面的交易数据：

   ```go
   proposal {  
       "transaction": transaction from above describe,
       "currentBlockNum": the current b2-node block number,
       "voteMax": how many b2-committer node need to vote,
       "txVoteNum": voteNums Step3,
       "resultVoteNum": voteNums Step8,
       "winAddress": which address has the entitlement to execute submitter to submit data(stateRoot and proof) to btc network
     "btcTxId": btc tx id which contains the stateRoot and proof (🚩这个是指什么)  
       "voteAdressList": already vote address list,
       "state": (PENDING, SUCCESS, TIMEOUT),
   }
   ```

   生成完proposal之后就记录这个proposalID

4. 从channel[proposalId]]中获取proposalId，并从b2节点api中搜索proposal状态。如果state为PENDING，则检查winAddress是否等于本地b2-committer地址，然后执行submitter向btc网络提交数据（stateRoot和proof） (相当于验证完2层交易后，2次最终性确认之后，再由一个共识网络进行小范围共识，最终提交到btc网络)

5. 从Step4中获取btcTxId，并将其放入通道[btcTxId]中，然后更新b2节点中的finalBatchNum （🚩这里的btcTxId是从第三步就已经出现了，是不是在那一步就已经提交到btc网络了？？）

6. 进入业务循环通道[btcTxId]，得到btcTxId，通过btcTxId查询btc网络的blockNum。如果已确认blockNum（大于7个块），则执行步骤7， （🚩这里的btcTxid 是否指的上面的）

7. the b2-committer which execute btc transaction and get the btcTxId update the proposal btcTxId. （🚩？）

8. 每个b2提交器节点循环它们的信道[proposealId]以检查btcTxId是否为空。如果不是，则检查btcTxId blockHeight并将事务提交给b2节点进行投票。resultVoteNum++。如果resultVoteNum>50%，则将状态更新为SUCCESS。

9. 通过通道[proposalId]中的proposalId循环检查proposal。如果proposal的状态仍然是PENDING并且（currentBlockNum-proposal.currentBlockNum）>10000，则将状态更新为TIMEOUT并再次执行步骤3。





## 大致代码设计

- 开启了go 循环专门获取b2 zknode 的最新高度 和btc 最新高度， 将zknode 的区块同步到数据库 
- 从同步到数据库的zkevm 区块，去过滤Polygonzkevm 合约的verifyBatch log，按照区块，把log全部转成事件存储到数据库
- 从数据库中查找proposal 相关记录根据lastFinalBatchNum(已经2次最终性确认，被验证过了)，最终是为了拿到VerifyRangBatchInfo（🚩，这部分代码涉及到zkevm 的设计原理），这个和proposalID进行挂钩，然后将stateroot 和proof (来源于VerifyRangBatchInfo)提交到B2-node (这是DA层). （配置的那个地址实际是b2-node上面的一个用户地址），如果没有问题，这个proposal相关的信息也会记录到数据库，**这个时候投票状态还没有记录。**，然后会有校验状态的进程去从b2node 查询，并更新到数据库。
- 当从b2node里检查 那个proposal的BitcoinTxHash 如果没有的话，说明还没有刻到比特币中，（这里有两个地址，一个是committer在BTC的地址，还有一个是 目标地址《是否是需要刻入的的地址》）
- 提交到taproot后，返回的bitcoinTxHash，将会写入到DA层，并和之前的proposalID 关联起来，然后同时也把这个proposal的某些字段更新到数据库（但是他后面又是在6个块确认后提交到DA，我不是很明白）