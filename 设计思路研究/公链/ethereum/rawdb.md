accessors_chain.go是go-ethereum/core/rawdb模块的一个文件，在v1.10.18版本中用于实现访问区块链数据的函数。以下是对该文件的详细分析：

accessors_chain.go文件定义了一系列函数，用于访问和获取区块链数据。这些函数大部分都是通过调用rawdb.go中的函数实现的，主要有以下几个函数：

1. GetBlockByHash：通过区块的哈希值获取对应的区块数据。
2. GetBlockByNumber：通过区块的高度获取对应的区块数据。
3. GetBlockHashesFromHash：获取从起始区块哈希值到结束区块哈希值之间所有区块的哈希值。
4. GetBlockHeaderByHash：通过区块的哈希值获取对应的区块头数据。
5. GetBlockHeaderByNumber：通过区块的高度获取对应的区块头数据。
6. GetCanonicalHash：获取指定高度的主链区块哈希值。
7. GetCanonicalHashes：获取指定高度范围内的主链区块哈希值。
8. GetHeaderByHash：通过区块的哈希值获取对应的区块头数据。
9. GetHeaderByNumber：通过区块的高度获取对应的区块头数据。
10. GetTd：获取指定区块哈希值的总难度。

这些函数的实现主要都是通过调用rawdb.go中的GetBlock、GetBlockHeader等函数实现的。其中，GetBlockByHash和GetBlockByNumber函数用于获取指定区块哈希值或高度的区块数据；GetBlockHeaderByHash和GetBlockHeaderByNumber函数用于获取指定区块哈希值或高度的区块头数据；GetCanonicalHash和GetCanonicalHashes函数用于获取指定高度的主链区块哈希值或者指定高度范围内的主链区块哈希值；GetHeaderByHash和GetHeaderByNumber函数用于获取指定区块哈希值或高度的区块头数据；GetTd函数用于获取指定区块哈希值的总难度。

除了上述函数外，accessors_chain.go文件还定义了一些辅助函数，如GetBlockByNumberOrHash、GetBlockHashes、GetHeader、GetTdByHash等。这些函数主要是为了方便访问和获取区块链数据而设计的。

总的来说，accessors_chain.go文件为其他模块提供了访问和获取区块链数据的接口，它们是以太坊系统的核心组件之一。这些函数的实现依赖于rawdb.go中提供的访问数据库的函数和数据存储结构，因此需要保证这些函数的正确性和可靠性，以确保整个以太坊系统的稳定运行。

需要注意的是，访问和获取区块链数据的效率是非常重要的。区块链数据的体量庞大，如果访问和获取数据的速度过慢，将会严重影响整个系统的性能。因此，在实现这些函数时需要考虑到效率和性能的问题。例如，在获取区块数据时，可以通过缓存、批量读取等方式来提高读取速度；在获取主链区块哈希值时，可以通过利用区块链的高度和哈希值的对应关系来加速查询。

除了效率和性能问题外，还需要考虑到数据一致性和可靠性的问题。区块链数据的正确性对整个系统的稳定运行至关重要，因此在访问和获取数据时需要保证数据的一致性和可靠性。例如，在获取区块数据时，需要确保读取的数据是正确的、完整的，并且数据的顺序是正确的。

总的来说，accessors_chain.go文件的实现需要兼顾效率、性能、一致性和可靠性等多个方面的要求，这是保证整个以太坊系统稳定运行的重要保障。



## ancient数据库

freezerdb 是以太坊的冻结数据库，它使用了两个子数据库：AncientStore 和 KeyValueStore

Freeze方法指定阈值90000个块，就会把区块塞到冷冻库

freezer 数据库是以太坊的一个优化功能，用于将旧的、不再需要访问的区块数据从 key-value 数据库中移除，并将其存储到独立的 freezer 数据库中。这些旧的区块在以太坊同步过程中不会被访问，但是在需要重新构建区块链状态时，这些旧的区块数据仍然非常重要。使用 freezer 数据库可以减少 key-value 数据库的大小，提高同步和查询性能，并且可以消除因 key-value 数据库大小过大而导致的同步失败问题。所以keyvalue数据库要和freeze数据库的数据能连起来。

`freezer_table.go` 是 Geth 中实现冷数据存储的一个重要文件，用于实现以太坊的冷数据存储机制。以下是对 `freezer_table.go` 的实现细节进行详细分析：

1. 数据结构

`freezer_table.go` 中定义了两个重要的数据结构：`FreezerTable` 和 `FreezerChunkStore`。

`FreezerTable` 是一个结构体，表示一个冷数据表。它包含了一个键值数据库句柄、数据表起始块号、每个冷数据块的大小、以及一个元数据缓存。元数据缓存用于缓存冷数据块的元数据，包括起始块号、结束块号、块的哈希值等信息。

`FreezerChunkStore` 是一个结构体，表示一个冷数据块存储器。它包含了一个键值数据库句柄、数据块的哈希值以及块的长度。它还包含了一个 `FreezerTable` 指针，用于表示该块所属的冷数据表。

1. 冷数据表初始化

在 `freezer_table.go` 中，冷数据表的初始化是通过 `NewFreezerTable` 函数实现的。该函数接收一个键值数据库句柄、起始块号、冷数据块大小等参数，并返回一个 `FreezerTable` 对象。

在初始化过程中，`NewFreezerTable` 函数会使用传入的键值数据库句柄创建一个名为 `freezer-metadata` 的命名空间，并将冷数据表的元数据存储在其中。然后，它会使用传入的起始块号和冷数据块大小计算每个冷数据块的起始块号和结束块号，并将这些元数据存储在元数据缓存中。

1. 冷数据的读写操作

在 `freezer_table.go` 中，冷数据的读写操作是通过 `FreezerTable`结构体中的 `Get` 和 `Put` 方法实现的。

`Get` 方法接收一个块号作为参数，并返回对应的冷数据块。具体实现过程如下：

首先，`Get` 方法会检查元数据缓存中是否有该块的元数据。如果有，它会使用元数据中的起始块号和结束块号计算出该块在数据库中的键名，并使用键值数据库句柄获取该键对应的数据块。

如果元数据缓存中没有该块的元数据，则会使用该块号和冷数据块大小计算出该块在数据库中的键名，并使用键值数据库句柄获取该键对应的数据块。此时，如果获取到了数据块，`Get` 方法会将该块的元数据存储到元数据缓存中。

`Put` 方法接收一个块号和一个冷数据块作为参数，并将该冷数据块存储到对应的冷数据块存储器中。具体实现过程如下：

首先，`Put` 方法会使用传入的块号计算出该块在数据库中的键名，并将该块存储到键值数据库句柄中。然后，它会将该块的元数据存储到元数据缓存中。

1. 冷数据块的哈希值计算

在 `freezer_table.go` 中，冷数据块的哈希值计算是通过 `FreezerChunkStore` 结构体的 `Hash` 方法实现的。

`Hash` 方法会读取存储在键值数据库句柄中的冷数据块，并使用 Keccak-256 算法计算出该块的哈希值。计算出的哈希值会被存储到 `FreezerChunkStore` 结构体中，并在之后的冷数据块读写操作中使用。

通过以上分析，我们可以了解到 Geth 中冷数据存储的实现原理和机制。冷数据表使用元数据缓存缓存冷数据块的元数据，以避免重复读取块的元数据。冷数据块存储器使用 Keccak-256 算法计算冷数据块的哈希值，并将哈希值存储到键值数据库中，以便在之后的读取操作中进行校验。在实际使用中，冷数据存储机制可以有效地减少以太坊节点的存储压力，提高节点的运行效率。



`freezer.go` 是 Geth 中实现冷数据管理的一个重要文件，用于实现以太坊的冷数据存储和管理机制。以下是对 `freezer.go` 的实现细节进行详细分析：

1. 数据结构

`freezer.go` 中定义了两个重要的数据结构：`Freezer` 和 `FreezerWriter`。

`Freezer` 是一个结构体，表示一个冷数据管理器。它包含了一个锁、一个键值数据库句柄、一个起始块号、一个冷数据块大小、以及一个冷数据表缓存。冷数据表缓存用于缓存冷数据表，以提高冷数据的读取效率。

`FreezerWriter` 是一个结构体，表示一个冷数据写入器。它包含了一个 `Freezer` 指针和一个写入队列。写入队列用于缓存待写入的冷数据块，以便在后续的写入操作中进行批量写入。

1. 冷数据表的创建和获取

在 `freezer.go` 中，冷数据表的创建和获取是通过 `Freezer` 结构体中的 `getTable` 方法实现的。

`getTable` 方法接收一个块号作为参数，并返回对应的冷数据表。具体实现过程如下：

首先，`getTable` 方法会检查冷数据表缓存中是否有该块所属的冷数据表。如果有，它会直接返回该冷数据表。

如果冷数据表缓存中没有该冷数据表，则会使用传入的块号和冷数据块大小计算出该块所属的冷数据表的起始块号，并使用 `FreezerTable.NewFreezerTable` 函数创建一个新的冷数据表。然后，它会将该冷数据表存储到冷数据表缓存中，并返回该冷数据表。

1. 冷数据的读写操作

在 `freezer.go`中，冷数据的读写操作是通过`Freezer`结构体中的`Read`和`Write` 方法实现的。

`Read` 方法接收一个块号作为参数，并返回对应的冷数据块。具体实现过程如下：

首先，`Read` 方法会获取对应冷数据块所属的冷数据表。然后，它会调用冷数据表的 `Get` 方法获取对应的冷数据块。

如果获取到了冷数据块，则会返回该冷数据块。

`Write` 方法接收一个块号和一个冷数据块作为参数，并将该冷数据块写入到对应的冷数据存储器中。具体实现过程如下：

首先，`Write` 方法会获取对应冷数据块所属的冷数据表。然后，它会将该冷数据块添加到冷数据写入器的写入队列中。

在冷数据写入器的后台协程中，会定期从写入队列中读取待写入的冷数据块，并将它们批量写入到对应的冷数据存储器中。写入操作会使用 `FreezerChunkStore.Put` 方法将冷数据块写入到键值数据库中，并使用 `FreezerTable.Put` 方法将冷数据块的元数据写入到冷数据表的元数据缓存中。

1. 冷数据的清理操作

在 `freezer.go` 中，冷数据的清理操作是通过 `Freezer` 结构体中的 `Prune` 方法实现的。

`Prune` 方法接收一个块号作为参数，并清除所有小于该块号的冷数据块。具体实现过程如下：

首先，`Prune` 方法会获取对应冷数据块所属的冷数据表，并使用冷数据表的 `Prune` 方法清除所有小于该块号的冷数据块。然后，它会从冷数据表缓存中移除该冷数据表，以释放缓存空间。

1. 冷数据的校验操作

在 `freezer.go` 中，冷数据的校验操作是通过 `Freezer` 结构体中的 `Verify` 方法实现的。

`Verify` 方法接收一个块号和一个块的哈希值作为参数，并校验该块的哈希值是否正确。具体实现过程如下：

首先，`Verify` 方法会获取对应冷数据块所属的冷数据表，并使用冷数据表的 `Get` 方法获取对应的冷数据块。然后，它会使用 Keccak-256 算法计算该冷数据块的哈希值，并将计算出的哈希值与传入的哈希值进行比较。

如果两个哈希值相等，则说明冷数据块的校验通过，`Verify` 方法会返回 `true`。否则，说明冷数据块的校验失败，`Verify` 方法会返回 `false`。

通过以上分析，我们可以了解到 Geth 中冷数据管理的实现原理和机制。冷数据管理器使用冷数据表缓存缓存冷数据表，以提高冷数据的读取效率。冷数据写入器使用批量写入的方式将冷数据块写入到冷数据存储器中，以提高写入效率。在实际使用中，冷数据管理机制可以有效地减少以太坊节点的存储压力，提高节点的运行效率。





##TODO

详细分析Ancient数据库