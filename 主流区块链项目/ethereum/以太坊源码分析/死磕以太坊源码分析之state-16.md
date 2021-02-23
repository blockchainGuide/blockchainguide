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

## 基础概念

### 状态机

以太坊的本质就是一个**基于交易的状态机(transaction-based state machine)**。在计算机科学中，一个 *状态机* 是指可以读取一系列的输入，然后根据这些输入，会转换成一个新的状态出来的东西。

我们从**创世纪状态(genesis state)**开始，在网络中还没有任何交易的时候产生状态。当第一个区块执行第一个交易时候开始产生状态，直到执行完N个交易，第一个区块的最终状态产生，第二个区块的第一笔交易执行后将会改变第一个区块链的最终状态，以此类推，从而产生最终的区块状态。

![image-20210112090020770](https://tva1.sinaimg.cn/large/008eGmZEgy1gmkmt3uuo1j31g20hatb4.jpg)

### 以太坊状态数据库

区块的状态数据并非保存在链上，而是将这些状态维护在默克尔压缩前缀树中，在区块链上仅记录对应的`Trie Root` 值。使用`LevelDB`维护树的持久化内容，而这个用来维护映射的数据库叫做 `StateDB`。

首先我们用一张图来大致了解一下`StateDB`：

![image-20210112165612767](https://tva1.sinaimg.cn/large/008eGmZEgy1gml0k9uv54j31c60g840z.jpg)

可以看到图中一共有两种状态，一个是世界状态`Trie`,一个是`storage Trie`,两者都是MPT树，世界状态包含了一个个的账户状态，账户状态通过以账户地址为键，维护在表示世界状态的树中，而每个账户状态中存储这账户存储树的`Root`。账户状态存储一下信息：

1. **nonce**: 表示此账户发出的交易数量
2. **balance**: 账户余额 
3. **storageRoot**:  账户存储树的Root根，用来存储合约信息
4. **codeHash**: 账户的 EVM 代码哈希值，当这个地址接收到一个消息调用时，这些代码会被执行; 它和其它字段不同，创建后不可更改。如果 codeHash 为空，则说明该账户是一个简单的外部账户，只存在 `nonce` 和 `balance`。

接下来将会分析State相关的一些类，着重关注`statedb.go、state_object.go、database.go`,其中涉及的Trie相关的代码可以参照：[死磕以太坊源码分析之MPT树-下](https://github.com/blockchainGuide/)

### 关键的数据结构

**Account**

Account存储的是账户状态信息。

```go
type Account struct {
	Nonce    uint64      //账户发出的交易数量
	Balance  *big.Int    // 账户的余额
	Root     common.Hash //账户存储树的Root根，用来存储合约信息
	CodeHash []byte      // 账户的 EVM 代码哈希值
}
```

**StateObject**

表示一个状态对象，可以从中获取到账户状态信息。

```go
type stateObject struct {
	address  common.Address
	addrHash common.Hash // 账户地址哈希
	data     Account
	db       *StateDB // 所属的StateDB
	dbErr error //VM不处理db层的错误，先记录下来，最后返回，只能保存1个错误，保存的第一个错误
	
  // Write caches.
	trie Trie // storage trie, 使用trie组织stateObj的数据
	code Code // 合约字节码，在加载代码时设置

	//将原始条目的存储高速缓存存储到dedup重写中，为每个事务重置
	originStorage Storage 

	//在整个块的末尾需要刷新到磁盘的存储条目
	pendingStorage Storage 

	//在当前事务执行中已修改的存储条目
	dirtyStorage Storage 

```

**StateDB**

用来存储状态对象。

```go
type StateDB struct {
  db   Database
	trie Trie // 当前所有账户组成的MPT树

	// 这几个相关账户状态修改
	stateObjects        map[common.Address]*stateObject // 存储缓存的账户状态信息
	stateObjectsPending map[common.Address]struct{}     // 状态对象已经完成但是还没有写入到Trie中
	stateObjectsDirty   map[common.Address]struct{}     // 在当前执行中修改的状态对象 ，用于后续commit 
}
```

> 三者之间的关系：
>
> StateDB->Trie->Account->stateObject
>
> **从StateDB中取出Trie根，根据地址从Trie树中获取账户的rlp编码数据，再进行解码成Account，然后根据Account生成stateObject**

## StateDB存储状态

StateDB读写状态主要关心以下几个文件：

- database.go
- state_object.go
- statedb.go

接下来分别介绍这么几个文件，相当关键。

### database.go

#### 根据世界状态root打开世界状态树

从`StateDB`中打开一个`Trie`大致经历以下过程：

> OpenTrie(root common.Hash)->NewSecure->New



#### 根据账户地址和 stoage root打开状态存储树

创建一个账户的存储Trie过程如下：

> OpenStorageTrie(addrHash, root common.Hash)->NewSecure-New



#### Account和StateObject

以太坊的账户分为普通账户和合约账户,以`Account`表示，`Account`是账户的数据，不包含账户地址，账户需要使用地址来表示，地址在`stateObject`中。

```go
type Account struct {
	Nonce    uint64
	Balance  *big.Int
	Root     common.Hash // 存储树的merkle树根 账户状态
	CodeHash []byte //合约账户专属，合约代码编译后的Hash值
}
```



```go
type stateObject struct {
  address  common.Address // 账户地址
	addrHash common.Hash // 账户地址哈希
	data     Account
	db       *StateDB // 所属的StateDB
  dbErr error //VM不处理db层的错误，先记录下来，最后返回，只能保存1个错误，保存存的第一个错误
	trie Trie // storage trie, 使用trie组织stateObj的数据
	code Code // 合约字节码，在加载代码时设置
	originStorage Storage //将原始条目的存储高速缓存存储到dedup重写中，为每个事务重置
	pendingStorage Storage //在整个块的末尾需要刷新到磁盘的存储条目
	dirtyStorage Storage //在当前事务执行中已修改的存储条目
}
```



#### 创建StateObject

创建状态对象会在两个地方进行调用：

1. 检索或者创建状态对象
2. 创建账户

最终都会去调用`createObject`创建一个新的状态对象。如果有一个现有的帐户给定的地址，老的将被覆盖并作为第二个返回值返回

```go
func (s *StateDB) createObject(addr common.Address) (newobj, prev *stateObject) {
	prev = s.getDeletedStateObject(addr)// 如果存在老的，获取用来以后删除掉

	newobj = newObject(s, addr, Account{})
	newobj.setNonce(0) 
	if prev == nil {
		s.journal.append(createObjectChange{account: &addr})
	} else {
		s.journal.append(resetObjectChange{prev: prev})
	}
	s.setStateObject(newobj)
	return newobj, prev
}
```



### state_object.go

`state_object.go`是很重要的文件，我们直接通过比较重要的函数来了解它。

#### 增加账户余额

```go
AddBalance->SetBalance
```

#### 将对象的存储树保存到db

主要就做了两件事：

1. *updateTrie将缓存的存储修改写入对象的存储Trie。* 
2. *将所有节点写入到trie的内存数据库中*

```go
func (s *stateObject) CommitTrie(db Database) error {
	s.updateTrie(db)
	...
	root, err := s.trie.Commit(nil)
	...
}
```

第一件事会在下面继续讲，第二件事可以参照我之前关于 [死磕以太坊源码分析之MPT树-下](https://github.com/blockchainGuide/)的讲解。

①：**将缓存的存储修改写入对象的存储Trie**

> 主要流程： 最终还是调用了trie.go的insert方法
>
> updateTrie->TryUpdate->insert

1. `s.finalise()` 将`dirtyStorage`中的所有数据移动到`pendingStorage`中
2. 根据账户哈希和账户`root`打开账户存储树
3. 将`key`与`trie`中的`value`关联，更新数据

```go
func (s *stateObject) updateTrie(db Database) Trie {
	s.finalise() ①
...
	
	tr := s.getTrie(db) ②
	for key, value := range s.pendingStorage {
		...
		if (value == common.Hash{}) {
			s.setError(tr.TryDelete(key[:]))
			continue
		}
	...
		s.setError(tr.TryUpdate(key[:], v)) ③
	}
...
}
```

整个核心也就是`updateTrie`，调用了`trie`的`insert`方法进行处理。

②：**将所有节点写入到trie的内存数据库，其key以sha3哈希形式存储**

> 流程：
>
> trie.Commit->t.trie.Commit->t.hashRoot

```go
func (t *SecureTrie) Commit(onleaf LeafCallback) (root common.Hash, err error) {
	if len(t.getSecKeyCache()) > 0 {
		t.trie.db.lock.Lock()
		for hk, key := range t.secKeyCache {
			t.trie.db.insertPreimage(common.BytesToHash([]byte(hk)), key)
		}
		t.trie.db.lock.Unlock()

		t.secKeyCache = make(map[string][]byte)
	}
	return t.trie.Commit(onleaf)
}
```

如果`KeyCache`中已经有了，直接插入到磁盘数据库，否则的话插入到`Trie`的内存数据库。



#### 将trie根设置为的当前根哈希

```go
func (s *stateObject) updateRoot(db Database) {
	s.updateTrie(db)
	if metrics.EnabledExpensive {
		defer func(start time.Time) { s.db.StorageHashes += time.Since(start) }(time.Now())
	}
	s.data.Root = s.trie.Hash()
}
```

方法也比较简单，底层调用`UpdateTrie`然后再更新`root`.

`State_object.go`的核心方法也就这么些内容。



### statedb.go

#### 创建账户

创建账户的核心就是创建状态对象，然后再初始化值。

```go
func (s *StateDB) CreateAccount(addr common.Address) {
	newObj, prev := s.createObject(addr)
	if prev != nil {
		newObj.setBalance(prev.data.Balance)
	}
}
```

```go
func (s *StateDB) createObject(addr common.Address) (newobj, prev *stateObject) {
	prev = s.getDeletedStateObject(addr) 

	newobj = newObject(s, addr, Account{})
	newobj.setNonce(0) 
	if prev == nil {
		s.journal.append(createObjectChange{account: &addr})
	} else {
		s.journal.append(resetObjectChange{prev: prev})
	}
	s.setStateObject(newobj)
	return newobj, prev
}
```

#### 删除、更新、获取状态对象

```go
func (s *StateDB) deleteStateObject(obj *stateObject) 
func (s *StateDB) updateStateObject(obj *stateObject) 
func (s *StateDB) getStateObject(obj *stateObject) {
```

这三个方法底层分别都是调用`Trie.TryDelete、Trie.TryUpdate、Trie.TryGet`方法来分别获取。

这里大致的讲一下`getStateObject`，代码如下：

```go
func (s *StateDB) getDeletedStateObject(addr common.Address) *stateObject {
	// Prefer live objects if any is available
	if obj := s.stateObjects[addr]; obj != nil {
		return obj
	}
	// Track the amount of time wasted on loading the object from the database
	if metrics.EnabledExpensive {
		defer func(start time.Time) { s.AccountReads += time.Since(start) }(time.Now())
	}
	// Load the object from the database
	enc, err := s.trie.TryGet(addr[:])
	if len(enc) == 0 {
		s.setError(err)
		return nil
	}
	var data Account
	if err := rlp.DecodeBytes(enc, &data); err != nil {
		log.Error("Failed to decode state object", "addr", addr, "err", err)
		return nil
	}
	// Insert into the live set
	obj := newObject(s, addr, data)
	s.setStateObject(obj)
	return obj
}
```

大致就做了以下几件事：

1. 先从`StateDB`中获取`stateObjects`,有的话就返回。
2. 如果没有的话就从`stateDB`的`trie`中获取账户状态数据，获取到`rlp`编码的数据之后，将其解码。
3. 根据状态数据`Account` 构造`stateObject`

#### 余额操作

余额的操作大致有添加、减少、和设定。我们就拿添加来分析：

根据地址获取`stateObject`，然后`addBalance`.

```go
func (s *StateDB) AddBalance(addr common.Address, amount *big.Int) {
	stateObject := s.GetOrNewStateObject(addr)
	if stateObject != nil {
		stateObject.AddBalance(amount)
	}
}
```



#### 储存快照和回退快照

```go
func (s *StateDB) Snapshot() int 
func (s *StateDB) RevertToSnapshot(revid int)
```

储存快照和回退快照，我们可以在提交交易的流程中找到：

```go
func (w *worker) commitTransaction(tx *types.Transaction, coinbase common.Address) ([]*types.Log, error) {
	snap := w.current.state.Snapshot()

	receipt, err := core.ApplyTransaction(w.chainConfig, w.chain, &coinbase, w.current.gasPool, w.current.state, w.current.header, tx, &w.current.header.GasUsed, *w.chain.GetVMConfig())
	if err != nil {
		w.current.state.RevertToSnapshot(snap)
		return nil, err
	}
	w.current.txs = append(w.current.txs, tx)
	w.current.receipts = append(w.current.receipts, receipt)

	return receipt.Logs, nil
}
```

首先我们会对当前状态进行快照，然后执行`ApplyTransaction`，如果在预执行交易的阶段出错了，那么会回退到备份的快照位置。之前的修改全部会回退。

#### 计算状态Trie的当前根哈希

计算状态Trie的当前根哈希是由`IntermediateRoot`来完成的。

①：**确定所有的脏存储状态（简单理解就是当前执行修改的所有对象）**

```go
func (s *StateDB) Finalise(deleteEmptyObjects bool) {
	for addr := range s.journal.dirties {
		obj, exist := s.stateObjects[addr]
		if !exist {
			continue
		}
		if obj.suicided || (deleteEmptyObjects && obj.empty()) {
			obj.deleted = true
		} else {
			obj.finalise()
		}
		s.stateObjectsPending[addr] = struct{}{}
		s.stateObjectsDirty[addr] = struct{}{}
	}
	s.clearJournalAndRefund()
}
```

其实这个跟`state_object`的`finalise`方法是一个方式，底层就是调用了`obj.finalise`将`dirty`状态的所有数据全部推入到`pending`中去，等待处理。

②：**处理stateObjectsPending中的数据**

先更新账户的`Root`根，然后再将将给定的对象写入`trie`。  

```go
for addr := range s.stateObjectsPending {
		obj := s.stateObjects[addr]
		if obj.deleted {
			s.deleteStateObject(obj)
		} else {
			obj.updateRoot(s.db)
			s.updateStateObject(obj)
		}
	}
```

  

#### 将状态写入底层内存Trie数据库

这部分功能由commit方法完成。

1.  计算状态Trie的当前根哈希
2. 将状态对象中的所有更改写入到存储树

第一步在上面已经讲过了，第二步的内容如下：

```go
for addr := range s.stateObjectsDirty {
		if obj := s.stateObjects[addr]; !obj.deleted {
			....
			if err := obj.CommitTrie(s.db); err != nil {
				return common.Hash{}, err
			}
		}
	}
```

核心就是`objectCommitTrie`,这也是上面`state_object`的内容。

总结流程如下：

> 1.IntermediateRoot
>
> 2.CommitTrie->updateTrie->trie.Commit->trie.db.insertPreimage(已经有了直接持久化到硬盘数据库)
>
> ​														  ->t.trie.Commit（没有就提交到存储树中）

最后看一下以太坊数据库的读写过程：

![image-20210113111013494](https://tva1.sinaimg.cn/large/008eGmZEgy1gmlw6izetsj31ha0oogom.jpg)

## 参考

> https://mindcarver.cn
>
> https://github.com/blockchainGuide
>
> https://www.jianshu.com/p/20d7f7c37b03
>
> https://hackernoon.com/getting-deep-into-ethereum-how-data-is-stored-in-ethereum-e3f669d96033
>
> https://web.xidian.edu.cn/qqpei/files/Blockchain/4_Data.pdf
>
> http://www.ltk100.com/article-112-1.html
>
> https://learnblockchain.cn/books/geth/part3/statedb.html


