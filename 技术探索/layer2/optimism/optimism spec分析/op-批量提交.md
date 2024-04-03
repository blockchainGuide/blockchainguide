# 批量提交者

批次提交者，也称为批处理者，是将 L2 排序器数据提交到 L1 的实体，以使其可供验证者使用。

数据事务的格式在[派生规范](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md)中定义：数据以与从数据派生到 L2 块相反的顺序从 L2 块构造。

时间、操作和交易签名是特定于实现的：任何数据都可以随时提交，但从验证者的角度来看，只有符合[派生规范规则的数据才是有效的。](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md)

最小的批处理程序实现可以定义为以下操作的循环：

1. 查看`unsafe`L2区块号是否超过`safe`区块号：`unsafe`需要提交数据。
2. 迭代所有不安全的 L2 块，跳过之前提交的任何块。
3. [打开一个通道，缓冲所有要提交的 L2 块数据，同时应用派生规范](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md)中定义的编码和压缩。
4. 从通道中拉出帧来填充数据事务，直到通道为空。
5. 将数据交易提交到L1

安全/不安全的 L2 视图不会在数据提交后立即更新，也不会在 L1 上确认时立即更新，因此可能需要特别注意不要重复数据提交。