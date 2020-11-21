## p2p源码目录概览

```go
p2p/
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

