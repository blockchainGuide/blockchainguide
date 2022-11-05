> https://godorz.info/2022/04/optimism-notes/

本文基于https://github.com/ethereum-optimism/optimism/commit/b1cc033b6d3827df47b606e3f89fdbf2fa2cccc4 撰写

> 学习目的，有意将此扩展方案适配到本公司基于以太坊更改的项目中以提高扩展性，更好的支持应用生态。

## Optimistic Rollup 概述

乐观汇总是一种第2层可扩展性技术，可在不牺牲安全性或分散性的情况下增加以太坊的计算和存储容量。交易数据在链上提交，但在链下执行。如果链外执行中存在错误，可以在链上提交错误证明，以纠正错误并保护用户资金。同样，除非有争议，否则你不会上法庭，除非有错误，否则你也不会在链上执行交易。



## 角色与行为

![image-20221026173531333](https://tva1.sinaimg.cn/large/008vxvgGgy1h7itqcdoxuj310w0h0wge.jpg)



































-----

## 组件







