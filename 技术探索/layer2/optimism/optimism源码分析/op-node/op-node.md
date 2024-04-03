## driver-sequencer 

sequencer 在完成构建L2区块的过程中，会调用engine, 会调用L1客户端获取所有指定L1 block （L2 Epoch，长达12秒，但是L2block 2秒一个，所以一个L1 大概出6个L2区块）的收据，从中获取depositTx的收据，

L2 block的第一笔交易必须是depositTxType,且必须存在一笔 L1 info deposit tx （这是一笔关于L1 block Info的交易，他是把block的信息放到了tx.data里了，按照DepositTx 格式，）

关于L2的系统配置，他是先在L1的合约上进行了设置，并触发了事件，然后sequencer通过扫描L1区块里面的交易事件，从而将配置导入到rollup node中。

实际PreparePayloadAttributes 就是从L1 获取上面的L1 block info 和L1上质押合约里的交易，两种都会转换成deposit tx,最终当作palayload 的属性。

forkchoiceUpdated 会调用op-geth 的方法，最终走到了buildPayload， 并没有进行共识	，实际都是走的那个prepare-fianliazeandassemble流程， 但是这个区块有被保存吗，状态信息呢

opnode 将状态转换完毕后，就该op-batcher干事了



## QA

1. 根据测试代码，还是不太能理解漂移窗口 ，如何查找l1origin 