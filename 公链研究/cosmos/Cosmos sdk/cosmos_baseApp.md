baseapp 实现了 cometBFT定义的接口：

```go
type Application interface {
	// Info/Query Connection
	Info(RequestInfo) ResponseInfo    // Return application info
	Query(RequestQuery) ResponseQuery // Query for state

	// Mempool Connection
	CheckTx(RequestCheckTx) ResponseCheckTx // Validate a tx for the mempool

	// Consensus Connection
	InitChain(RequestInitChain) ResponseInitChain // Initialize blockchain w validators/other info from CometBFT
	PrepareProposal(RequestPrepareProposal) ResponsePrepareProposal
	ProcessProposal(RequestProcessProposal) ResponseProcessProposal
	BeginBlock(RequestBeginBlock) ResponseBeginBlock // Signals the beginning of a block
	DeliverTx(RequestDeliverTx) ResponseDeliverTx    // Deliver a tx for full processing
	EndBlock(RequestEndBlock) ResponseEndBlock       // Signals the end of a block, returns changes to the validator set
	Commit() ResponseCommit                          // Commit the state and return the application Merkle root hash

	// State Sync Connection
	ListSnapshots(RequestListSnapshots) ResponseListSnapshots                // List available snapshots
	OfferSnapshot(RequestOfferSnapshot) ResponseOfferSnapshot                // Offer a snapshot to the application
	LoadSnapshotChunk(RequestLoadSnapshotChunk) ResponseLoadSnapshotChunk    // Load a snapshot chunk
	ApplySnapshotChunk(RequestApplySnapshotChunk) ResponseApplySnapshotChunk // Apply a shapshot chunk
}
```



baseapp的结构：

```go
type BaseApp struct {
	// initialized on creation
	logger            log.Logger
	name              string                      // application name from abci.BlockInfo
	db                dbm.DB                      // common DB backend
	cms               storetypes.CommitMultiStore // Main (uncached) state
	qms               storetypes.MultiStore       // Optional alternative multistore for querying only.
	storeLoader       StoreLoader                 // function to handle store loading, may be overridden with SetStoreLoader()
	grpcQueryRouter   *GRPCQueryRouter            // router for redirecting gRPC query calls
	msgServiceRouter  *MsgServiceRouter           // router for redirecting Msg service messages
	interfaceRegistry codectypes.InterfaceRegistry
	txDecoder         sdk.TxDecoder // unmarshal []byte into sdk.Tx
	txEncoder         sdk.TxEncoder // marshal sdk.Tx into []byte

	mempool            mempool.Mempool            // application side mempool
	anteHandler        sdk.AnteHandler            // ante handler for fee and auth
	postHandler        sdk.PostHandler            // post handler, optional, e.g. for tips
	initChainer        sdk.InitChainer            // initialize state with validators and state blob
	beginBlocker       sdk.BeginBlocker           // logic to run before any txs
	processProposal    sdk.ProcessProposalHandler // the handler which runs on ABCI ProcessProposal
	prepareProposal    sdk.PrepareProposalHandler // the handler which runs on ABCI PrepareProposal
	endBlocker         sdk.EndBlocker             // logic to run after all txs, and to determine valset changes
	prepareCheckStater sdk.PrepareCheckStater     // logic to run during commit using the checkState
	precommiter        sdk.Precommiter            // logic to run during commit using the deliverState
	addrPeerFilter     sdk.PeerFilter             // filter peers by address and port
	idPeerFilter       sdk.PeerFilter             // filter peers by node ID
	fauxMerkleMode     bool                       // if true, IAVL MountStores uses MountStoresDB for simulation speed.

	// manages snapshots, i.e. dumps of app state at certain intervals
	snapshotManager *snapshots.Manager

	// volatile states:
	//
	// checkState is set on InitChain and reset on Commit
	// deliverState is set on InitChain and BeginBlock and set to nil on Commit
	checkState           *state // for CheckTx
	deliverState         *state // for DeliverTx
	processProposalState *state // for ProcessProposal
	prepareProposalState *state // for PrepareProposal

	// an inter-block write-through cache provided to the context during deliverState
	interBlockCache storetypes.MultiStorePersistentCache

	// paramStore is used to query for ABCI consensus parameters from an
	// application parameter store.
	paramStore ParamStore

	// The minimum gas prices a validator is willing to accept for processing a
	// transaction. This is mainly used for DoS and spam prevention.
	minGasPrices sdk.DecCoins

	// initialHeight is the initial height at which we start the baseapp
	initialHeight int64

	// flag for sealing options and parameters to a BaseApp
	sealed bool

	// block height at which to halt the chain and gracefully shutdown
	haltHeight uint64

	// minimum block time (in Unix seconds) at which to halt the chain and gracefully shutdown
	haltTime uint64

	// minRetainBlocks defines the minimum block height offset from the current
	// block being committed, such that all blocks past this offset are pruned
	// from CometBFT. It is used as part of the process of determining the
	// ResponseCommit.RetainHeight value during ABCI Commit. A value of 0 indicates
	// that no blocks should be pruned.
	//
	// Note: CometBFT block pruning is dependant on this parameter in conjunction
	// with the unbonding (safety threshold) period, state pruning and state sync
	// snapshot parameters to determine the correct minimum value of
	// ResponseCommit.RetainHeight.
	minRetainBlocks uint64

	// application's version string
	version string

	// application's protocol version that increments on every upgrade
	// if BaseApp is passed to the upgrade keeper's NewKeeper method.
	appVersion uint64

	// recovery handler for app.runTx method
	runTxRecoveryMiddleware recoveryMiddleware

	// trace set will return full stack traces for errors in ABCI Log field
	trace bool

	// indexEvents defines the set of events in the form {eventType}.{attributeKey},
	// which informs CometBFT what to index. If empty, all events will be indexed.
	indexEvents map[string]struct{}

	// streamingManager for managing instances and configuration of ABCIListener services
	streamingManager storetypes.StreamingManager

	chainID string
}
```