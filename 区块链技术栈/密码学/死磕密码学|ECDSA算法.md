> 死磕密码学|ECDSA算法
> 文章资料代码请star https://github.com/blockchainGuide/

## 生成签名

 假设 Alice 希望对消息$m$进行签名，所采用的椭圆曲线参数为$D=(p,a,b,G,n,h)$，对应的密钥对为$(k,Q)$，其中为公钥$Q$，$k$为私钥。

  Alice 将按如下步骤进行签名：

  1. 产生一个随机数$d$，$1 \leq d \leq n-1$.  （签名算法首先生成一个临时私公钥对，该临时密钥对用于计算 r 和 s 值。）
  2. 计算$dG=(x_1,y_1)$，将$x_1$化为整数$\overline{x_1}$.
  3. 计算$r=\overline{x_1} \ mod \  n$，若$r=0$，则转向第1步.     （r 值为临时公钥的坐标 x 值）
  4. 计算 $d^{-1} \ mod \ n$
  5. 计算哈希值$H(m)$，并将得到的比特串转化为整数 e
  6. 计算$s=d^{-1}(e+kr) \ mod \ n$，若$s=0$，则转向第1步.
  7. $(r,s)$即为 Alice 对消息的签名.

## 椭圆曲线签名验证

  为验证 Alice 对消息 m 的签名$(r,s)$，Bob 需要得到 Alice 所用的椭圆曲线参数$D=(p,a,b,G,n,h)$以及 Alice 的公钥 Q。

  步骤如下：

  1. 验证 r 和 s 是区间$[1,n-1]$上的整数.
  2. 计算$H(m)$并将其转化为整数 e.
  3. 计算$w=s^{-1} \ mod \ n$
  4. 计算$u_1=ew \ mod \ n$以及$u_2=rw \ mod \ n$
  5. 计算$X=(x_1,y_1)=u_1G+u_2Q$
  6. 若$X=O$，则拒绝签名，否则将 X 的$x$坐标$x_1$转化为整数，并计$\overline{x_1}$算$v=\overline{x_1} \ mod \ n$
  7. 当且仅当$v=r$时，签名通过验证.

## 椭圆曲线签名正确性

  要证明$v=r$，只需要证明$X=dG$即可。

  证明步骤：

  令：$C=u_1G + u_2Q = u_1G+u_2kG=(u_1+u_2k)G$

  将$u_1$、$u_2$带入：$C=(ew+rwk)G=(e+rk)wG=(e+rk)s^{-1}G$

  由$s=d^{-1}(e+kr) \mod  n$得出$s^{-1}=d(e+kr)^{-1} \mod n$，带入： $C=(e+kr)d(d+kr)^{-1}G = dG$

## 使用案例

```go
func main() {
	//生成签名----
	//声明明文
	message := []byte("hello world")
	//生成私钥
	privateKey, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	//生成公钥
	pub := privateKey.PublicKey
	//将明文散列
	digest := sha256.Sum256(message)
	//生成签名
	r, s, _ := ecdsa.Sign(rand.Reader, privateKey, digest[:])
	//设置私钥的参数类型为曲线类型
	param := privateKey.Curve.Params()
	//获得私钥byte长度
	curveOrderByteSize := param.P.BitLen() / 8
	//获得签名返回值的字节
	rByte, sByte := r.Bytes(), s.Bytes()
	//创建数组
	signature := make([]byte, curveOrderByteSize*2)
	//通过数组保存了签名结果的返回值
	copy(signature[curveOrderByteSize-len(rByte):], rByte)
	copy(signature[curveOrderByteSize*2-len(sByte):], sByte)

	//验证----
	//将明文做hash散列，为了验证的内容对比
	digest = sha256.Sum256(message)
	curveOrderByteSize = pub.Curve.Params().P.BitLen() / 8
	//创建两个整形对象
	r, s = new(big.Int), new(big.Int)
	//设置证书值
	r.SetBytes(signature[:curveOrderByteSize])
	s.SetBytes(signature[curveOrderByteSize:])

	//验证
	e := ecdsa.Verify(&pub, digest[:], r, s)
	if e == true {
		fmt.Println("success")
	} else {
		fmt.Println("failed")
	}
}
```

## 参考

>  https://mindcarver.cn 
>
> https://juejin.cn/post/6844903671411671047
>
> https://juejin.cn/post/6844903882343071758
>
> http://read.pudn.com/downloads54/sourcecode/windows/188357/ECDSA%E7%AE%97%E6%B3%95%E5%AE%9E%E7%8E%B0%E5%8F%8A%E5%85%B6%E5%AE%89%E5%85%A8%E6%80%A7%E5%88%86%E6%9E%90.pdf
>
> https://zhuanlan.zhihu.com/p/94852431





