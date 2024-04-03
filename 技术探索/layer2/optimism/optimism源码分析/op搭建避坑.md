1. 第二必须安装一致的node js 版本
2. 要加一个systemblock 啥玩意
3. 不知道官方文档为什么把pnpm 换成了yarn，变成yarn isntall 和yarnbuild ,我的linux 照样用ppm
4. https://learnblockchain.cn/question/3799 https://github.com/Qingquan-Li/blog/issues/131 ,解决mac 老是send requset 超时的问题，mac 默认终端不会被代理，需要设置下。



```go
## 
hash :0x6ff874e39a51c87799b068ac5bdf027cca565d27f539b01c1691b430be65c1b7
number:1000
timestamp:1691995518
```

4. 有个json文件需要加systemConfigStartBlock，我写的就是上面的1000，有哥们说直接写0 
5. direnv allow . 不起作用，直接source吧



## op-geth 启动

```shell
 ./build/bin/geth   --datadir ./datadir   --http   --http.corsdomain="*"   --http.vhosts="*"   --http.addr=0.0.0.0   --http.api=web3,debug,eth,txpool,net,engine   --ws   --ws.addr=0.0.0.0   --ws.port=8888   --ws.origins="*"   --ws.api=debug,eth,txpool,net,engine   --syncmode=full   --gcmode=archive   --nodiscover   --maxpeers=0   --networkid=42069   --authrpc.vhosts="*"   --authrpc.addr=0.0.0.0   --authrpc.port=8551   --authrpc.jwtsecret=./jwt.txt   --rollup.disabletxpoolgossip=true   --password=./datadir/password   --allow-insecure-unlock   --mine   --miner.etherbase=$SEQ_ADDR   --unlock=$SEQ_ADDR
```



## op-node

```shell
./bin/op-node \
	--l2=http://localhost:8551 \
	--l2.jwt-secret=./jwt.txt \
	--sequencer.enabled \
	--sequencer.l1-confs=3 \
	--verifier.l1-confs=3 \
	--rollup.config=./rollup.json \
	--rpc.addr=0.0.0.0 \
	--rpc.port=8547 \
	--p2p.disable \
	--rpc.enable-admin \
	--p2p.sequencer.key=$SEQ_KEY \
	--l1=$L1_RPC \
	--l1.rpckind=$RPC_KIND
```

