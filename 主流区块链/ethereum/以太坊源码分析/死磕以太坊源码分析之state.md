> 死磕以太坊源码分析之state
>
> 配合以下代码进行阅读：https://github.com/blockchainGuide/
>
> 希望读者在阅读过程中发现问题可以及时评论哦，大家一起进步。



## 源码目录

```
｜-database.go 底层的存储设计
｜-dump.go  用来dumpstateDB数据
｜-iterator.go，用来遍历Trie
｜-journal.go，用来记录状态的改变
｜-state_object.go 通过state object操作账户值，并将修改后的storage trie写入数据库
｜-statedb.go，以太坊整个的状态
｜-sync.go，用来和downloader结合起来同步state
```



## 整体架构





## 重点关注







## 参考

> https://mindcarver.cn
>
> https://github.com/blockchainGuide