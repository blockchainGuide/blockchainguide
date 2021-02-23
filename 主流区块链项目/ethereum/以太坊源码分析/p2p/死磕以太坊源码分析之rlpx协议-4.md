> 死磕以太坊源码分析之rlpx协议


本文主要参考自eth官方文档：[rlpx协议](https://github.com/blockchainGuide)

## 符号

- `X || Y`：表示X和Y的串联
- `X ^ Y`： X和Y按位异或
- `X[:N]`：X的前N个字节
- `[X, Y, Z, ...]`：[X, Y, Z, ...]的RLP递归编码
- `keccak256(MESSAGE)`：以太坊使用的keccak256哈希算法
- `ecies.encrypt(PUBKEY, MESSAGE, AUTHDATA)`：RLPx使用的非对称身份验证加密函数     AUTHDATA是身份认证的数据，并非密文的一部分     但是AUTHDATA会在生成消息tag前，写入HMAC-256哈希函数
- `ecdh.agree(PRIVKEY, PUBKEY)`：是PRIVKEY和PUBKEY之间的椭圆曲线Diffie-Hellman协商函数

--------

## ECIES加密

ECIES (Elliptic Curve Integrated Encryption Scheme) 非对称加密用于RLPx握手。RLPx使用的加密系统：

- 椭圆曲线secp256k1基点`G`
- `KDF(k, len)`：密钥推导函数 NIST SP 800-56 Concatenation
- `MAC(k, m)`：HMAC函数，使用了SHA-256哈希
- `AES(k, iv, m)`：AES-128对称加密函数，CTR模式

假设Alice想发送加密消息给Bob，并且希望Bob可以用他的静态私钥`kB`解密。Alice知道Bob的静态公钥`KB`。

Alice为了对消息`m`进行加密：

1. 生成一个随机数`r`并生成对应的椭圆曲线公钥`R = r * G`
2. 计算共享密码`S = Px`，其中 `(Px, Py) = r * KB`
3. 推导加密及认证所需的密钥`kE || kM = KDF(S, 32)`以及随机向量`iv`
4. 使用AES加密 `c = AES(kE, iv, m)`
5. 计算MAC校验 `d = MAC(keccak256(kM), iv || c)`
6. 发送完整密文`R || iv || c || d`给Bob

Bob对密文`R || iv || c || d`进行解密：

1. 推导共享密码`S = Px`, 其中`(Px, Py) = r * KB = kB * R`
2. 推导加密认证用的密钥`kE || kM = KDF(S, 32)`
3. 验证MAC`d = MAC(keccak256(kM), iv || c)`
4. 获得明文`m = AES(kE, iv || c)`

-----

## 节点身份

所有的加密操作都基于**secp256k1**椭圆曲线。每个节点维护一个静态的**secp256k1**私钥。建议该私钥只能进行手动重置（例如删除文件或数据库条目）。

-----

## 握手流程

RLPx连接基于TCP通信，并且每次通信都会生成随机的临时密钥用于加密和验证。生成临时密钥的过程被称作“握手” (handshake)，握手在发起端（initiator, 发起TCP连接请求的节点）和接收端（recipient, 接受连接的节点）之间进行。

1. 发起端向接收端发起TCP连接，发送`auth`消息
2. 接收端接受连接，解密、验证`auth`消息（检查recovery of signature == `keccak256(ephemeral-pubk)`）
3. 接收端通过`remote-ephemeral-pubk` 和 `nonce`生成`auth-ack`消息
4. 接收端推导密钥，发送首个包含[Hello](https://github.com/ethereum/devp2p/blob/master/rlpx.md#hello-0x00)消息的数据帧 (frame)
5. 发起端接收到`auth-ack`消息，导出密钥
6. 发起端发送首个加密后的数据帧，包含发起端[Hello](https://github.com/ethereum/devp2p/blob/master/rlpx.md#hello-0x00)消息
7. 接收端接收并验证首个加密后的数据帧
8. 发起端接收并验证首个加密后的数据帧
9. 如果两边的首个加密数据帧的MAC都验证通过，则加密握手完成

如果首个数据帧的验证失败，则任意一方都可以断开连接。

### 握手消息

**发送端：**

```go
auth = auth-size || enc-auth-body
auth-size = size of enc-auth-body, encoded as a big-endian 16-bit integer
auth-vsn = 4
auth-body = [sig, initiator-pubk, initiator-nonce, auth-vsn, ...]
enc-auth-body = ecies.encrypt(recipient-pubk, auth-body || auth-padding, auth-size)
auth-padding = arbitrary data
```

**接收端：**

```go
ack = ack-size || enc-ack-body
ack-size = size of enc-ack-body, encoded as a big-endian 16-bit integer
ack-vsn = 4
ack-body = [recipient-ephemeral-pubk, recipient-nonce, ack-vsn, ...]
enc-ack-body = ecies.encrypt(initiator-pubk, ack-body || ack-padding, ack-size)
ack-padding = arbitrary data
```

实现必须忽略`auth-vsn` 和 `ack-vsn`中的所有不匹配。

实现必须忽略`auth-body` 和 `ack-body`中的所有额外列表元素。

握手消息互换后，密钥生成：

```go
static-shared-secret = ecdh.agree(privkey, remote-pubk)
ephemeral-key = ecdh.agree(ephemeral-privkey, remote-ephemeral-pubk)
shared-secret = keccak256(ephemeral-key || keccak256(nonce || initiator-nonce))
aes-secret = keccak256(ephemeral-key || shared-secret)
mac-secret = keccak256(ephemeral-key || aes-secret)
```

## 帧结构

握手后所有的消息都按帧 (frame) 传输。一帧数据携带属于某一功能的一条加密消息。

分帧传输的主要目的是在单一连接上实现可靠的支持多路复用协议。其次，因数据包分帧，为消息认证码产生了适当的分界点，使得加密流变得简单了。通过握手生成的密钥对数据帧进行加密和验证。

帧头提供关于消息大小和消息源功能的信息。填充字节用于防止缓存区不足，使得帧组件按指定区块字节大小对齐。

```go
frame = header-ciphertext || header-mac || frame-ciphertext || frame-mac
header-ciphertext = aes(aes-secret, header)
header = frame-size || header-data || header-padding
header-data = [capability-id, context-id]
capability-id = integer, always zero
context-id = integer, always zero
header-padding = zero-fill header to 16-byte boundary
frame-ciphertext = aes(aes-secret, frame-data || frame-padding)
frame-padding = zero-fill frame-data to 16-byte boundary
```

-----

## MAC

RLPx中的消息认证 (Message authentication) 使用了两个keccak256状态，分别用于两个传输方向。`egress-mac`和`ingress-mac`分别代表发送和接收状态，每次发送或者接收密文，其状态都会更新。初始握手后，MAC状态初始化如下:

**发送端：**

```go
egress-mac = keccak256.init((mac-secret ^ recipient-nonce) || auth)
ingress-mac = keccak256.init((mac-secret ^ initiator-nonce) || ack)
```

**接收端：**

```go
egress-mac = keccak256.init((mac-secret ^ initiator-nonce) || ack)
ingress-mac = keccak256.init((mac-secret ^ recipient-nonce) || auth)
```

当发送一帧数据时，通过即将发送的数据更新`egress-mac`状态，然后计算相应的MAC值。通过将帧头与其对应MAC值的加密输出异或来进行更新。这样做是为了确保对明文MAC和密文执行统一操作。所有的MAC值都以明文发送。

```
header-mac-seed = aes(mac-secret, keccak256.digest(egress-mac)[:16]) ^ header-ciphertext
egress-mac = keccak256.update(egress-mac, header-mac-seed)
header-mac = keccak256.digest(egress-mac)[:16]
```

**计算 `frame-mac`**

```
egress-mac = keccak256.update(egress-mac, frame-ciphertext)
frame-mac-seed = aes(mac-secret, keccak256.digest(egress-mac)[:16]) ^ keccak256.digest(egress-mac)[:16]
egress-mac = keccak256.update(egress-mac, frame-mac-seed)
frame-mac = keccak256.digest(egress-mac)[:16]
```

只要发送者和接受者按相同方式更新`egress-mac`和`ingress-mac`，并且在ingress帧中比对`header-mac` 和 `frame-mac`的值，就能对ingress帧中的MAC值进行校验。这一步应当在解密`header-ciphertext` 和 `frame-ciphertext`之前完成。

----

## 功能消息

初始握手后的所有消息均与“功能”相关。单个RLPx连接上就可以同时使用任何数量的功能。

功能由简短的ASCII名称和版本号标识。连接两端都支持的功能在隶属于“ p2p”功能的[Hello](https://github.com/ethereum/devp2p/blob/master/rlpx.md#hello-0x00)消息中进行交换，p2p功能需要在所有连接中都可用。

### 消息编码

初始Hello消息编码如下：

```
frame-data = msg-id || msg-data
frame-size = length of frame-data, encoded as a 24bit big-endian integer
```

其中，`msg-id`是标识消息的由RLP编码的整数，`msg-data`是包含消息数据的RLP列表。

Hello之后的所有消息均使用Snappy算法压缩。请注意，压缩消息的`frame-size`指`msg-data`压缩前的大小。消息的压缩编码为:

```
frame-data = msg-id || snappyCompress(msg-data)
frame-size = length of (msg-id || msg-data) encoded as a 24bit big-endian integer
```

## 基于`msg-id`的复用

frame中虽然支持`capability-id`，但是在本RLPx版本中并没有将该字段用于不同功能之间的复用（当前版本仅使用msg-id来实现复用）。

每种功能都会根据需要分配尽可能多的msg-id空间。所有这些功能所需的msg-id空间都必须通过静态指定。在连接和接收[Hello](https://github.com/ethereum/devp2p/blob/master/rlpx.md#hello-0x00)消息时，两端都具有共享功能（包括版本）的对等信息，并且能够就msg-id空间达成共识。

msg-id应当大于0x11(0x00-0x10保留用于“ p2p”功能）。

-----

## p2p功能

所有连接都具有“p2p”功能。初始握手后，连接的两端都必须发送[Hello](https://github.com/ethereum/devp2p/blob/master/rlpx.md#hello-0x00)或[Disconnect](https://github.com/ethereum/devp2p/blob/master/rlpx.md#disconnect-0x01)消息。在接收到Hello消息后，会话就进入激活状态，并且可以开始发送其他消息。由于前向兼容性，实现必须忽略协议版本中的所有差异。与处于较低版本的节点通信时，实现应尝试靠近该版本。

任何时候都可能会收到[Disconnect](https://github.com/ethereum/devp2p/blob/master/rlpx.md#disconnect-0x01)消息。

### Hello (0x00)

```
[protocolVersion: P, clientId: B, capabilities, listenPort: P, nodeKey: B_64, ...]
```

握手完成后，双方发送的第一包数据。在收到Hello消息前，不能发送任何其他消息。实现必须忽略Hello消息中所有其他列表元素，因为可能会在未来版本中用到。

- `protocolVersion`当前p2p功能版本为第5版
- `clientId`表示客户端软件身份，人类可读字符串, 比如"Ethereum(++)/1.0.0“
- `capabilities`支持的子协议列表，名称及其版本：`[[cap1, capVersion1], [cap2, capVersion2], ...]`
- `listenPort`节点的收听端口 (位于当前连接路径的接口)，0表示没有收听
- `nodeId`secp256k1的公钥，对应节点私钥

### Disconnect (0x01)

```
[reason: P]
```

通知节点断开连接。收到该消息后，节点应当立即断开连接。如果是发送，正常的主机会给节点2秒钟读取时间，使其主动断开连接。

`reason` 一个可选整数，表示断开连接的原因：

| Reason | Meaning                                                      |
| ------ | ------------------------------------------------------------ |
| `0x00` | Disconnect requested                                         |
| `0x01` | TCP sub-system error                                         |
| `0x02` | Breach of protocol, e.g. a malformed message, bad RLP, ...   |
| `0x03` | Useless peer                                                 |
| `0x04` | Too many peers                                               |
| `0x05` | Already connected                                            |
| `0x06` | Incompatible P2P protocol version                            |
| `0x07` | Null node identity received - this is automatically invalid  |
| `0x08` | Client quitting                                              |
| `0x09` | Unexpected identity in handshake                             |
| `0x0a` | Identity is the same as this node (i.e. connected to itself) |
| `0x0b` | Ping timeout                                                 |
| `0x10` | Some other reason specific to a subprotocol                  |

### Ping (0x02)

```
[]
```

要求节点立即进行[Pong](https://github.com/ethereum/devp2p/blob/master/rlpx.md#pong-0x03)回复。

### Pong (0x03)

```
[]
```

回复节点的[Ping](https://github.com/ethereum/devp2p/blob/master/rlpx.md#ping-0x02)包。

-----

## 源码分析

### 主要功能

#### 返回传输对象

> 返回一个transport对象,连接持续5秒

```go
// handshakeTimeout 5
func newRLPX(fd net.Conn) transport {
....
}
```

#### 读取消息

> 返回Msg对象,调用读写器的ReadMsg,连接持续30秒

```go
func (t *rlpx) ReadMsg() (Msg, error) {
  ..
	t.fd.SetReadDeadline(time.Now().Add(frameReadTimeout))
}
```

#### 写入消息

> 调用读写器的WriteMsg写信息,连接持续20秒

```go
func (t *rlpx) WriteMsg(msg Msg) error {
  ...
	t.fd.SetWriteDeadline(time.Now().Add(frameWriteTimeout))
}
```

#### 协议版本握手

> 协议握手,输入输出均是protoHandshake对象,包含了版本号、名称、容量、端口号、ID和一个扩展属性,握手时会对这些信息进行验证

#### 加密握手

> 握手时主动发起者叫**initiator**
>
> 接收方叫**receiver**
>
> 分别对应两种处理方式**initiatorEncHandshake**和receiverEncHandshake
>
> 两种处理方式成功以后都会得到一个**secrets**对象,保存了共享密钥信息,它会跟原有的**net.Conn**对象一起生成一个帧处理器:**rlpxFrameRW**
>
> 握手双方使用到的信息有:各自的公私钥地址对**(iPrv,iPub,rPrv,rPub)**、各自生成的随机公私钥对**(iRandPrv,iRandPub,rRandPrv,rRandPub)**、各自生成的临时随机数**(initNonce,respNonce).**
>  其中i开头的表示发起方**(initiator)**信息,r开头的表示接收方**(receiver)**信息.

```go
func (t *rlpx) doEncHandshake(prv *ecdsa.PrivateKey, dial *ecdsa.PublicKey) (*ecdsa.PublicKey, error) {
	var (
		sec secrets
		err error
	)
	if dial == nil {
		sec, err = receiverEncHandshake(t.fd, prv) // 接收者
	} else {
		sec, err = initiatorEncHandshake(t.fd, prv, dial) //主动发起者
	}
...
	t.rw = newRLPXFrameRW(t.fd, sec)
	t.wmu.Unlock()
	return sec.Remote.ExportECDSA(), nil
}
```

这里我们就讲解一下主动握手部分源码`initiatorEncHandshake`：

①：初始化握手对象

```GO
h := &encHandshake{initiator: true, remote: ecies.ImportECDSAPublic(remote)}
```

②：生成验证信息

```go
authMsg, err := h.makeAuthMsg(prv) 
```

```GO
func (h *encHandshake) makeAuthMsg(prv *ecdsa.PrivateKey) (*authMsgV4, error) {
	// 生成己方随机数initNonce
	h.initNonce = make([]byte, shaLen)
	_, err := rand.Read(h.initNonce)
...
	}
// 生成随机的一组公私钥对
	h.randomPrivKey, err = ecies.GenerateKey(rand.Reader, crypto.S256(), nil)
...
	}
	// 生成静态共享秘密token(用己方私钥和对方公钥进行有限域乘法)
	token, err := h.staticSharedSecret(prv)
	...
	}
//  和己方随机数异或后用随机生成的私钥签名
	signed := xor(token, h.initNonce)
	signature, err := crypto.Sign(signed, h.randomPrivKey.ExportECDSA())
...
	}
...
	return msg, nil
}
```

③：封包,将验证信息和握手进行rlp编码并拼接前缀信息

```go
authPacket, err := sealEIP8(authMsg, h)
```

④：通过conn发送消息

```go
conn.Write(authPacket)
```

⑤：处理接收的信息,得到响应包

> `readHandshakeMsg`比较简单。 首先用一种格式尝试解码。如果不行就换另外一种。应该是一种兼容性的设置。 基本上就是使用自己的私钥进行解码然后调用rlp解码成结构体。 
>
> 结构体的描述就是下面的authRespV4,里面最重要的就是对端的随机公钥。 双方通过自己的私钥和对端的随机公钥可以得到一样的共享秘密。 而这个共享秘密是第三方拿不到的

```GO
	authRespMsg := new(authRespV4)
	authRespPacket, err := readHandshakeMsg(authRespMsg, encAuthRespLen, prv, conn)
```

⑥：填充响应的respNonce(对方随机数,生成共享私钥用)和remoteRandomPub(对方的随机公钥)

```GO
 h.handleAuthResp(authRespMsg)
```

⑦：将请求包和响应包封装成共享秘密(secrets)

```go
h.secrets(authPacket, authRespPacket)
```

到此RLPX 相关的比较重要的内容就解读差不多了。

------

## 参考

> https://github.com/blockchainGuide/blockchainguide  ☆ ☆ ☆ ☆ ☆
>
> https://mindcarver.cn/  ☆ ☆ ☆ ☆ ☆
>
> https://github.com/ethereum/devp2p/blob/master/rlpx.md 

