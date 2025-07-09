## 介绍

Celestia is a data availability (DA) layer that provides a scalable solution to the [data availability problem](https://coinmarketcap.com/alexandria/article/what-is-data-availability). 

## 关键功能

### [data availability sampling](https://blog.celestia.org/celestia-mvp-release-data-availability-sampling-light-clients) (DAS)

Celestia使用二维Reed-Solomon编码方案对块数据进行编码：将每个块数据拆分为k×k个块，排列在k×k矩阵中，并通过多次Reed-Solmon编码将奇偶校验数据扩展为2k×2k扩展矩阵

然后，为扩展矩阵的行和列计算4k个单独的Merkle根；这些Merkle根的Merkle根部被用作块报头中的块数据承诺。

### [Namespaced Merkle trees](https://github.com/celestiaorg/nmt) (NMTs).



## QA

