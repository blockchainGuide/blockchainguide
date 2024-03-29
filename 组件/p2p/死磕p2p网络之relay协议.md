> 当一个对等端无法监听公共地址时，它可以拨出到中继对等端，这将保持长期连接打开。其他对等方将能够使用p2p电路地址通过中继对等方拨号，从而将流量转发到其目的地。
>
> 中继连接是端到端加密的，这意味着充当中继的对等方无法读取或篡改流经连接的任何流量



## 应用场景

假设我有一个对等ID为QmAlice的对等。我想把我的地址给我的朋友QmBob，但我有一个NAT，它不允许任何人直接给我打电话。

![image-20221018142037486](https://tva1.sinaimg.cn/large/008vxvgGgy1h79f7koq8qj30z90u0mzn.jpg)

节点A位于NAT和/或防火墙后面，例如通过AutoNAT服务检测到的。

因此，节点A请求与中继R进行预约，即节点A请求中继R代表其侦听传入连接。

节点B希望与节点a建立连接。鉴于节点**a不公布任何直接地址**，而只公布中继地址，节点B连接到中继R，要求中继R中继到a的连接。

中继R将连接请求转发到节点A，并最终中继A和B发送的所有数据。



## 参考

https://github.com/libp2p/specs/blob/master/relay/circuit-v2.md