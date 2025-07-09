> 死磕以太坊源码分析之MPT树-下
>
> 文章以及资料请查看：https://github.com/blockchainGuide/
>



[上篇](https://github.com/blockchainGuide/)主要介绍了以太坊中的MPT树的原理，这篇主要会对MPT树涉及的源码进行拆解分析。`trie`模块主要有以下几个文件：

```GO
|-encoding.go 主要讲编码之间的转换
|-hasher.go 实现了从某个结点开始计算子树的哈希的功能
|-node.go 定义了一个Trie树中所有结点的类型和解析的代码
|-sync.go 实现了SyncTrie对象的定义和所有方法
|-iterator.go 定义了所有枚举相关接口和实现
|-secure_trie.go 实现了SecureTrie对象
|-proof.go 为key构造一个merkle证明
|-trie.go Trie树的增删改查
|-database.go 对内存中的trie树节点进行引用计数
```



## 实现概览



### encoding.go

这个主要是讲三种编码（`KEYBYTES encoding`、`HEX encoding`、`COMPACT encoding`）的实现与转换，`trie`中全程都需要用到这些，该文件中主要实现了如下功能：

1. hex编码转换为Compact编码：`hexToCompact()`
2. Compact编码转换为hex编码：`compactToHex()`
3. keybytes编码转换为Hex编码：`keybytesToHex()`
4. hex编码转换为keybytes编码：`hexToKeybytes()`
5. 获取两个字节数组的公共前缀的长度：`prefixLen()`

```go
func hexToCompact(hex []byte) []byte {
    terminator := byte(0)
    if hasTerm(hex) { //检查是否有结尾为0x10 => 16
        terminator = 1 //有结束标记16说明是叶子节点
        hex = hex[:len(hex)-1] //去除尾部标记
    }
    buf := make([]byte, len(hex)/2+1) // 字节数组
    
    buf[0] = terminator << 5 // 标志byte为00000000或者00100000
    //如果长度为奇数，添加奇数位标志1，并把第一个nibble字节放入buf[0]的低四位
    if len(hex)&1 == 1 {
        buf[0] |= 1 << 4 // 奇数标志 00110000
        buf[0] |= hex[0] // 第一个nibble包含在第一个字节中 0011xxxx
        hex = hex[1:]
    }
    //将两个nibble字节合并成一个字节
    decodeNibbles(hex, buf[1:])
    return buf
  
```

```go
//compact编码转化为Hex编码
func compactToHex(compact []byte) []byte {
    base := keybytesToHex(compact)
    base = base[:len(base)-1]
     // apply terminator flag
    // base[0]包括四种情况
    // 00000000 扩展节点偶数位
    // 00000001 扩展节点奇数位
    // 00000010 叶子节点偶数位
    // 00000011 叶子节点奇数位

    // apply terminator flag
    if base[0] >= 2 {
       //如果是叶子节点，末尾添加Hex标志位16
        base = append(base, 16)
    }
    // apply odd flag
    //如果是偶数位，chop等于2，否则等于1
    chop := 2 - base[0]&1
    return base[chop:]
}
```

```go
//compact编码转化为Hex编码
func compactToHex(compact []byte) []byte {
    base := keybytesToHex(compact)
    base = base[:len(base)-1]
     // apply terminator flag
    // base[0]包括四种情况
    // 00000000 扩展节点偶数位
    // 00000001 扩展节点奇数位
    // 00000010 叶子节点偶数位
    // 00000011 叶子节点奇数位

    // apply terminator flag
    if base[0] >= 2 {
       //如果是叶子节点，末尾添加Hex标志位16
        base = append(base, 16)
    }
    // apply odd flag
    //如果是偶数位，chop等于2，否则等于1
    chop := 2 - base[0]&1
    return base[chop:]
}
```

```go
// 将十六进制的bibbles转成key bytes，这只能用于偶数长度的key
func hexToKeybytes(hex []byte) []byte {
    if hasTerm(hex) {
        hex = hex[:len(hex)-1]
    }
    if len(hex)&1 != 0 {
        panic("can't convert hex key of odd length")
    }
    key := make([]byte, (len(hex)+1)/2)
    decodeNibbles(hex, key)
    return key
}


```

```go
// 返回a和b的公共前缀的长度
func prefixLen(a, b []byte) int {
    var i, length = 0, len(a)
    if len(b) < length {
        length = len(b)
    }
    for ; i < length; i++ {
        if a[i] != b[i] {
            break
        }
    }
    return i
}


```



### node.go

#### 四种节点

node 接口分四种实现: fullNode，shortNode，valueNode，hashNode，其中只有 fullNode 和 shortNode 可以带有子节点。

```go
type (
	fullNode struct {
		Children [17]node // 分支节点
		flags    nodeFlag
	}
	shortNode struct { //扩展节点
		Key   []byte
		Val   node //可能指向叶子节点，也可能指向分支节点。
		flags nodeFlag
	}
	hashNode  []byte
	valueNode []byte // 叶子节点值，但是该叶子节点最终还是会包装在shortNode中
)
```



### trie.go

Trie对象实现了MPT树的所有功能，包括(key, value)对的增删改查、计算默克尔哈希，以及将整个树写入数据库中。



### iterator.go

`nodeIterator`提供了遍历树内部所有结点的功能。其结构如下：此结构体是在`trie.go`定义的

```go
type nodeIterator struct {
	trie.NodeIterator
	t   *odrTrie
	err error
}
```

里面包含了一个接口`NodeIterator`，它的实现则是由`iterator.go`来提供的，其方法如下：

```go
func (it *nodeIterator) Next(descend bool) bool 
func (it *nodeIterator) Hash() common.Hash 
func (it *nodeIterator) Parent() common.Hash 
func (it *nodeIterator) Leaf() bool 
func (it *nodeIterator) LeafKey() []byte 
func (it *nodeIterator) LeafBlob() []byte 
func (it *nodeIterator) LeafProof() [][]byte 
func (it *nodeIterator) Path() []byte {}
func (it *nodeIterator) seek(prefix []byte) error 
func (it *nodeIterator) peek(descend bool) (*nodeIteratorState, *int, []byte, error) 
func (it *nodeIterator) nextChild(parent *nodeIteratorState, ancestor common.Hash) (*nodeIteratorState, []byte, bool) 
func (it *nodeIterator) push(state *nodeIteratorState, parentIndex *int, path []byte) 
func (it *nodeIterator) pop() 
```

`NodeIterator`的核心是`Next`方法，每调用一次这个方法，NodeIterator对象代表的当前节点就会更新至下一个节点，当所有结点遍历结束，`Next`方法返回`false`。



生成NodeIterator结口的方法有以下3种：

**①：Trie.NodeIterator(start []byte)**

通过`start`参数指定从哪个路径开始遍历，如果为`nil`则从头到尾按顺序遍历。



**②：NewDifferenceIterator(a, b NodeIterator)**

当调用`NewDifferenceIterator(a, b NodeIterator)`后，生成的`NodeIterator`只遍历存在于 b 但不存在于 a 中的结点。



**③：NewUnionIterator(iters []NodeIterator)**

当调用`NewUnionIterator(its []NodeIterator)`后，生成的`NodeIterator`遍历的结点是所有传入的结点的合集。



### database.go

`Database`是`trie`模块对真正数据库的缓存层，其目的是对缓存的节点进行引用计数，从而实现区块的修剪功能。主要方法如下：

```go
func NewDatabase(diskdb ethdb.KeyValueStore) *Database
func NewDatabaseWithCache(diskdb ethdb.KeyValueStore, cache int) *Database 
func (db *Database) DiskDB() ethdb.KeyValueReader
func (db *Database) InsertBlob(hash common.Hash, blob []byte)
func (db *Database) insert(hash common.Hash, blob []byte, node node)
func (db *Database) insertPreimage(hash common.Hash, preimage []byte)
func (db *Database) node(hash common.Hash) node
func (db *Database) Node(hash common.Hash) ([]byte, error)
func (db *Database) preimage(hash common.Hash) ([]byte, error)
func (db *Database) secureKey(key []byte) []byte
func (db *Database) Nodes() []common.Hash
func (db *Database) Reference(child common.Hash, parent common.Hash)
func (db *Database) Dereference(root common.Hash)
func (db *Database) dereference(child common.Hash, parent common.Hash)
func (db *Database) Cap(limit common.StorageSize) error
func (db *Database) Commit(node common.Hash, report bool) error
```

### security_trie.go

可以理解为加密了的`trie`的实现，`ecurity_trie`包装了一下`trie`树， 所有的`key`都转换成`keccak256`算法计算的`hash`值。同时在数据库里面存储`hash`值对应的原始的`key`。
但是官方在代码里也注释了，这个代码不稳定，除了测试用例，别的地方并没有使用该代码。

### proof.go

- Prove()：根据给定的`key`，在`trie`中，将满足`key`中最大长度前缀的路径上的节点都加入到`proofDb`（队列中每个元素满足：未编码的hash以及对应`rlp`编码后的节点）
- VerifyProof()：验证`proffDb`中是否存在满足输入的`hash`，和对应key的节点，如果满足，则返回`rlp`解码后的该节点。



## 实现细节

### Trie对象的增删改查

①：**Trie树的初始化**

如果`root`不为空，就通过`resolveHash`来加载整个`Trie`树，如果为空，就新建一个`Trie`树。

```go
func New(root common.Hash, db *Database) (*Trie, error) {
	if db == nil {
		panic("trie.New called without a database")
	}
	trie := &Trie{
		db: db,
	}
	if root != (common.Hash{}) && root != emptyRoot {
		rootnode, err := trie.resolveHash(root[:], nil)
		if err != nil {
			return nil, err
		}
		trie.root = rootnode
	}
	return trie, nil
}
```

②：**Trie树的插入**

首先Trie树的插入是个递归调用的过程，它会从根开始找，一直找到合适的位置插入。

```go
func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error)
```

参数说明：

- n: 当前要插入的节点
- prefix: 当前已经处理完的**key**(节点共有的前缀)
- key: 未处理完的部分**key**，完整的`key = prefix + key`
- value：需要插入的值

返回值说明：

- bool : 操作是否改变了**Trie**树(**dirty**)
- Node :插入完成后的子树的根节点

接下来就是分别对`shortNode`、`fullNode`、`hashNode`、`nil` 几种情况进行说明。

**2.1：节点为nil**

空树直接返回`shortNode`， 此时整颗树的根就含有了一个`shortNode`节点。

```GO
case nil:
		return true, &shortNode{key, value, t.newFlag()}, nil
```

**2.2 ：节点为shortNode**

- 首先计算公共前缀，如果公共前缀就等于`key`，那么说明这两个`key`是一样的，如果`value`也一样的(`dirty == false`)，那么返回错误。

- 如果没有错误就更新`shortNode`的值然后返回

- 如果公共前缀不完全匹配，那么就需要把公共前缀提取出来形成一个独立的节点(扩展节点),扩展节点后面连接一个`branch`节点，`branch`节点后面看情况连接两个`short`节点。

- 首先构建一个branch节点(branch := &fullNode{flags: t.newFlag()}),然后再branch节点的Children位置调用t.insert插入剩下的两个short节点

```go
matchlen := prefixLen(key, n.Key)
		if matchlen == len(n.Key) {
			dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
			if !dirty || err != nil {
				return false, n, err
			}
			return true, &shortNode{n.Key, nn, t.newFlag()}, nil
		}
		branch := &fullNode{flags: t.newFlag()}
		var err error
		_, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
		if err != nil {
			return false, nil, err
		}
		_, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
		if err != nil {
			return false, nil, err
		}
		if matchlen == 0 {
			return true, branch, nil
    }
		return true, &shortNode{key[:matchlen], branch, t.newFlag()}, nil
```

**2.3: 节点为fullNode**

节点是`fullNode`(也就是分支节点)，那么直接往对应的孩子节点调用`insert`方法,然后把对应的孩子节点指向新生成的节点。  

```go
dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
		if !dirty || err != nil {
			return false, n, err
		}
		n = n.copy()
		n.flags = t.newFlag()
		n.Children[key[0]] = nn
		return true, n, nil
```

**2.4: 节点为hashnode**

暂时还在数据库中的节点，先调用 `t.resolveHash(n, prefix)`来加载到内存，然后调用`insert`方法来插入。

```go
rn, err := t.resolveHash(n, prefix)
		if err != nil {
			return false, nil, err
		}
		dirty, nn, err := t.insert(rn, prefix, key, value)
		if !dirty || err != nil {
			return false, rn, err
		}
		return true, nn, nil
```

③：**Trie树查询值**

其实就是根据输入的`hash`，找到对应的叶子节点的数据。主要看`TryGet`方法。

参数：

- `origNode`：当前查找的起始**node**位置
- `key`：输入要查找的数据的**hash**
- `pos`：当前**hash**匹配到第几位

```go
func (t *Trie) tryGet(origNode node, key []byte, pos int) (value []byte, newnode node, didResolve bool, err error) {
	switch n := (origNode).(type) {
	case nil: //表示当前trie是空树
		return nil, nil, false, nil
	case valueNode: ////这就是我们要查找的叶子节点对应的数据
		return n, n, false, nil
	case *shortNode: ////在叶子节点或者扩展节点匹配
		if len(key)-pos < len(n.Key) || !bytes.Equal(n.Key, key[pos:pos+len(n.Key)]) {
			return nil, n, false, nil
		}
		value, newnode, didResolve, err = t.tryGet(n.Val, key, pos+len(n.Key))
		if err == nil && didResolve {
			n = n.copy()
			n.Val = newnode
		}
		return value, n, didResolve, err
	case *fullNode://在分支节点匹配
		value, newnode, didResolve, err = t.tryGet(n.Children[key[pos]], key, pos+1)
		if err == nil && didResolve {
			n = n.copy()
			n.Children[key[pos]] = newnode
		}
		return value, n, didResolve, err
	case hashNode: //说明当前节点是轻节点，需要从db中获取
		child, err := t.resolveHash(n, key[:pos])
		if err != nil {
			return nil, n, true, err
		}
		value, newnode, _, err := t.tryGet(child, key, pos)
		return value, newnode, true, err
...
}
```

`didResolve`用于判断`trie`树是否会发生变化，`tryGet()`只是用来获取数据的，当`hashNode`去`db`中获取该`node`值后需要更新现有的trie，`didResolve`就会发生变化。其他就是基本的递归查找树操作。

④：**Trie树更新值**

更新值，其实就是调用insert方法进行操作。

到此Trie树的增删改查就讲解的差不多了。



### 将节点写入到Trie的内存数据库

如果要把节点写入到内存数据库，需要序列化，可以先去了解下以太坊的Rlp编码。这部分工作由`trie.Commit()`完成，当`trie.Commit(nil)`，会执行序列化和缓存等操作，序列化之后是使用的`Compact Encoding`进行编码，从而达到节省空间的目的。

```go
func (t *Trie) Commit(onleaf LeafCallback) (root common.Hash, err error) {
	if t.db == nil {
		panic("commit called on trie with nil database")
	}
	hash, cached, err := t.hashRoot(t.db, onleaf)
	if err != nil {
		return common.Hash{}, err
	}
	t.root = cached
	return common.BytesToHash(hash.(hashNode)), nil
}
```

上述代码大概讲了这些：

- 每次执行`Commit()`，该trie的`cachegen`就会加 1
- `Commit()`方法返回的是`trie.root`所指向的`node`的`hash`（未编码）
- 其中的`hashRoot()`方法目的是`返回trie.root所指向的node的hash`以及`每个节点都带有各自hash的trie树的root`。

```go
//为每个node生成一个hash
func (t *Trie) hashRoot(db *Database, onleaf LeafCallback) (node, node, error) {
	if t.root == nil {
		return hashNode(emptyRoot.Bytes()), nil, nil
	}
	h := newHasher(onleaf)
	defer returnHasherToPool(h)
	return h.hash(t.root, db, true) //为每个节点生成一个未编码的hash
}
```

而`hashRoot`的核心方法就是 `h.hash`，它返回了头节点的`hash`以及每个子节点都带有`hash`的头节点（Trie.root指向它），大致做了以下几件事：

①：*如果我们不存储节点，而只是哈希，则从缓存中获取数据*

```go
if hash, dirty := n.cache(); hash != nil {
		if db == nil {
			return hash, n, nil
		}
		if !dirty {
			switch n.(type) {
			case *fullNode, *shortNode:
				return hash, hash, nil
			default:
				return hash, n, nil
			}
		}
	}
```

②：*递归调用`h.hashChildren`，求出所有的子节点的`hash`值，再把原有的子节点替换成现在子节点的`hash`值*

**2.1:如果节点是`shortNode`**

首先把`collapsed.Key从Hex Encoding` 替换成 `Compact Encoding`, 然后递归调用`hash`方法计算子节点的`hash`和`cache`，从而把子节点替换成了子节点的`hash`值

```go
collapsed, cached := n.copy(), n.copy()
		collapsed.Key = hexToCompact(n.Key)
		cached.Key = common.CopyBytes(n.Key)

		if _, ok := n.Val.(valueNode); !ok {
			collapsed.Val, cached.Val, err = h.hash(n.Val, db, false)
			if err != nil {
				return original, original, err
			}
		}
		return collapsed, cached, nil
```

**2.2:节点是fullNode**

 遍历每个子节点，把子节点替换成子节点的`Hash`值，否则的化这个节点没有`children`。直接返回。

```go
		collapsed, cached := n.copy(), n.copy()

		for i := 0; i < 16; i++ {
			if n.Children[i] != nil {
				collapsed.Children[i], cached.Children[i], err = h.hash(n.Children[i], db, false)
				if err != nil {
					return original, original, err
				}
			}
		}
		cached.Children[16] = n.Children[16]
		return collapsed, cached, nil
```

③：*存储节点n的哈希值，如果我们指定了存储层，它会写对应的键/值对*

store()方法主要就做了两件事：

- `rlp`序列化`collapsed`节点并将其插入db磁盘中
- 生成当前节点的`hash`
- 将节点哈希插入`db`

**3.1：空数据或者hashNode，则不处理**

```go
if _, isHash := n.(hashNode); n == nil || isHash {
		return n, nil
	}
```

**3.2:生成节点的RLP编码**

```go
h.tmp.Reset()                                 // 缓存初始化
	if err := rlp.Encode(&h.tmp, n); err != nil { //将当前node序列化
		panic("encode error: " + err.Error())
	}
	if len(h.tmp) < 32 && !force {
		return n, nil // Nodes smaller than 32 bytes are stored inside their parent 编码后的node长度小于32，若force为true，则可确保所有节点都被编码
	}
//长度过大的，则都将被新计算出来的hash取代
	hash, _ := n.cache() //取出当前节点的hash
	if hash == nil {
		hash = h.makeHashNode(h.tmp) //生成哈希node
	}
```

**3.3:将Trie节点合并到中间内存缓存中**

```go
hash := common.BytesToHash(hash)
		db.lock.Lock()
		db.insert(hash, h.tmp, n)
		db.lock.Unlock()
		// Track external references from account->storage trie
		//跟踪帐户->存储Trie中的外部引用
		if h.onleaf != nil {
			switch n := n.(type) {
			case *shortNode:
				if child, ok := n.Val.(valueNode); ok {  //指向的是分支节点
					h.onleaf(child, hash) //用于统计当前节点的信息，比如当前节点有几个子节点，当前有效的节点数
				}
			case *fullNode:
				for i := 0; i < 16; i++ {
					if child, ok := n.Children[i].(valueNode); ok {
						h.onleaf(child, hash)
					}
				}
			}
		}
```

到此为止将节点写入到`Trie`的内存数据库就已经完成了。

*如果觉得文章不错可以关注公众号：**区块链技术栈**，详细的所有以太坊源码分析文章内容以及代码资料都在其中。*

### Trie树缓存机制

`Trie`树的结构里面有两个参数， 一个是`cachegen`,一个是`cachelimit`。这两个参数就是`cache`控制的参数。 `Trie`树每一次调用`Commit`方法，会导致当前的`cachegen`增加1。

```go
func (t *Trie) Commit(onleaf LeafCallback) (root common.Hash, err error) {
   ...
    t.cachegen++
   ...
}
```

然后在`Trie`树插入的时候，会把当前的`cachegen`存放到节点中。

```go
func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
            ....
            return true, &shortNode{n.Key, nn, t.newFlag()}, nil
}
func (t *Trie) newFlag() nodeFlag {
    return nodeFlag{dirty: true, gen: t.cachegen}
}

```

如果 `trie.cachegen - node.cachegen > cachelimit`，就可以把节点从内存里面拿掉。 也就是说节点经过几次`Commit`，都没有修改，那么就把节点从内存里面干掉。 只要`trie`路径上新增或者删除一个节点，整个路径的节点都需要重新实例化，也就是节点中的`nodeFlag`被初始化了。都需要重新更新到`db`磁盘。

拿掉节点过程在 `hasher.hash`方法中， 这个方法是在`commit`的时候调用。如果方法的`canUnload`方法调用返回真，那么就拿掉节点，如果只返回了`hash`节点，而没有返回`node`节点，这样节点就没有引用，不久就会被gc清除掉。 节点被拿掉之后，会用一个`hashNode`节点来表示这个节点以及其子节点。 如果后续需要使用，可以通过方法把这个节点加载到内存里面来。

```go
func (h *hasher) hash(n node, db *Database, force bool) (node, node, error) {
   	....
       // 从缓存中卸载节点。它的所有子节点将具有较低或相等的缓存世代号码。
       cacheUnloadCounter.Inc(1)
  ...
}
```



## 参考&总结

这部分重要的内容也就上面讲述的，主要集中在`Trie`上面，如果有不对的地方，可以及时指正哦。

> https://mindcarver.cn/about/
>
> https://github.com/blockchainGuide/blockchainguide

