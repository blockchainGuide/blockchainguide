## Q1 如何保证验证者节点之间的稳定连接，同时也能连接其他类型节点

### 基于Role的DHT设计

1. 基于Role的哈希函数

   设计一个角色敏感的哈希函数，使得具有相同角色的节点在DHT空间中更有可能彼此靠近。例如，可以为验证节点和普通节点分别使用不同的哈希函数前缀

   ```go
   func roleSensitiveHash(role Role, data []byte) []byte {
       var prefix []byte
       if role == ValidatorNode {
           prefix = []byte("validator")
       } else {
           prefix = []byte("regular")
       }
   
       hashData := append(prefix, data...)
       hash := sha256.Sum256(hashData)
       return hash[:]
   }
   ```

2. 自定义节点发现策略

   ```go
   // 自定义节点发现策略
       discoveryOptions := []discovery.Option{
           discovery.Filter(func(info *peer.AddrInfo) bool {
               // 检查节点角色是否与当前节点匹配
               return checkRoleMatch(info, nodeRole)
           }),
       }
   ```

------

### 验证节点专属子网络

1. 创建子网络DHT 实例：

   为验证节点专属子网络创建一个单独的 DHT 实例。这个实例可以使用与主网络相同的 DHT 协议，但应该具有不同的网络标识符，以避免与主网络混淆。这可以通过为子网络 DHT 实例选择一个特殊的网络前缀来实现。

   ```go
   validatorDHT, err := dht.New(ctx, validatorHost, dht.ProtocolPrefix("/validator-dht"))
   if err != nil {
       // 处理错误
   }
   ```

2. 验证节点加入子网络

   当一个验证节点加入网络时，它应首先加入主网络，然后尝试加入验证节点专属子网络。为此，验证节点需要知道其他验证节点的一些初始地址。这可以通过预先配置的引导节点列表或其他节点发现方法来实现

   ```go
   // 加入主网络
   bootstrap(ctx, host, mainDHT)
   
   // 如果是验证节点，还要加入验证节点专属子网络
   if isValidator {
       bootstrap(ctx, validatorHost, validatorDHT)
   }
   ```

3. 优先与子网络中的验证节点建立连接

   在验证节点子网络中，验证节点应优先与其他验证节点建立连接。这可以通过在子网络 DHT 中进行节点发现并尝试与找到的验证节点建立连接

   ```go
   func connectToValidators(ctx context.Context, host host.Host, validatorDHT *dht.IpfsDHT) {
       for {
           // 查找子网络中的其他验证节点
           peerChan, err := validatorDHT.FindPeers(ctx, "validator-subnet")
           if err != nil {
               // 处理错误
           }
   
           // 尝试与找到的验证节点建立连接
           for peer := range peerChan {
               if peer.ID != host.ID() && host.Network().Connectedness(peer.ID) != network.Connected {
                   _, err := host.Network().DialPeer(ctx, peer.ID)
                   if err != nil {
                       // 处理错误
                   }
               }
           }
   
           // 在下一轮查找之前等待一段时间
           time.Sleep(time.Minute)
       }
   }
   ```

4. 允许验证节点与普通节点建立连接：

   4.1 定义最大普通节点连接数

   为验证节点定义一个最大普通节点连接数。这个值可以根据您的需求进行调整，以控制验证节点与普通节点之间的连接数量。

   4.2 定期查找普通节点：

   验证节点应定期在主网络 DHT 中查找普通节点。为了避免与其他验证节点建立连接，验证节点应首先检查找到的节点的角色，然后仅连接到普通节点

   ```go
   func findRegularNodes(ctx context.Context, dht *dht.IpfsDHT) (<-chan peer.AddrInfo, error) {
       peerChan, err := dht.FindPeers(ctx, "regular-nodes")
       if err != nil {
           return nil, err
       }
   
       filteredChan := make(chan peer.AddrInfo)
       go func() {
           defer close(filteredChan)
           for peer := range peerChan {
               role, err := getPeerRole(ctx, dht, peer.ID)
               if err == nil && role == RegularNode {
                   filteredChan <- peer
               }
           }
       }()
   
       return filteredChan, nil
   }
   ```

   4.3 维护与普通节点的连接

   验证节点需要维护与普通节点的连接。当连接数低于最大普通节点连接数时，验证节点应尝试与新的普通节点建立连接。如果连接数超过最大值，验证节点可以断开一些连接，以便与其他普通节点建立连接。

   ```go
   func maintainRegularConnections(ctx context.Context, host host.Host, dht *dht.IpfsDHT) {
       for {
           regularNodes, err := findRegularNodes(ctx, dht)
           if err != nil {
               // 处理错误
           }
   
           regularConnections := 0
           for peer := range regularNodes {
               if host.Network().Connectedness(peer.ID) == network.Connected {
                   regularConnections++
               } else if regularConnections < maxRegularConnections {
                   _, err := host.Network().DialPeer(ctx, peer.ID)
                   if err != nil {
                       // 处理错误
                   } else {
                       regularConnections++
                   }
               }
           }
   
           // 在下一轮查找之前等待一段时间
           time.Sleep(time.Minute)
       }
   }
   ```

   