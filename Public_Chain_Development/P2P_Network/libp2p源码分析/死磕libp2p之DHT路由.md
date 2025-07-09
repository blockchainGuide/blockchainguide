## 接口定义

```go
//ContentRouting是间接寻址的值提供程序层。它用于查找有关谁拥有什么内容的信息。
//内容由CID（内容标识符）标识，CID对哈希进行编码
//以防将来使用。
```

```go
type ContentRouting interface {
	// 将给定的cid添加到内容路由系统
	Provide(context.Context, cid.Cid, bool) error

	// 搜索能够提供给定key的peers
	FindProvidersAsync(context.Context, cid.Cid, int) <-chan peer.AddrInfo
}
```



```go
type PeerRouting interface {
	// 搜索具有给定ID的peer，并返回相关地址信息
	FindPeer(context.Context, peer.ID) (peer.AddrInfo, error)
}
```



```go
type Routing interface {
	ContentRouting // 内容路由
	PeerRouting    // 节点路由
	ValueStore     // 基础存取接口
	Bootstrap(context.Context) error // // 允许caller提示路由系统进入Boostrapped状态并保持在那里。它不是同步调用。
}
```

```go
type PubKeyFetcher interface {
	// GetPublicKey returns the public key for the given peer.
	GetPublicKey(context.Context, peer.ID) (ci.PubKey, error)
}
```



