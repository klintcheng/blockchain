# 1. blockchain

<!-- TOC -->

- [1. blockchain](#1-blockchain)
    - [1.1. 简介](#11-%E7%AE%80%E4%BB%8B)
        - [1.1.1. BlockChain](#111-blockchain)
        - [1.1.2. blockNode](#112-blocknode)
    - [1.2. blockchain创建流程](#12-blockchain%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B)
        - [1.2.1. Checkpoint索引](#121-checkpoint%E7%B4%A2%E5%BC%95)
        - [1.2.2. New BlockChain](#122-new-blockchain)
        - [1.2.3. initChainState](#123-initchainstate)
            - [1.2.3.1. createChainState](#1231-createchainstate)
            - [1.2.3.2. 加载链状态](#1232-%E5%8A%A0%E8%BD%BD%E9%93%BE%E7%8A%B6%E6%80%81)
        - [1.2.4. UpgradeDbBuckets](#124-upgradedbbuckets)
        - [1.2.5. IndexManager初始化](#125-indexmanager%E5%88%9D%E5%A7%8B%E5%8C%96)
            - [1.2.5.1. transaction index](#1251-transaction-index)
            - [1.2.5.2. address index](#1252-address-index)
            - [1.2.5.3. committed filter index](#1253-committed-filter-index)
        - [1.2.6. initThresholdCaches](#126-initthresholdcaches)
            - [1.2.6.1. isCurrent](#1261-iscurrent)

<!-- /TOC -->

## 1.1. 简介

本章大致看下创建blockchain对象过程。

### 1.1.1. BlockChain

```go
// BlockChain provides functions for working with the bitcoin block chain.
// It includes functionality such as rejecting duplicate blocks, ensuring blocks
// follow all rules, orphan handling, checkpoint handling, and best chain
// selection with reorganization.
type BlockChain struct {
    // The following fields are set when the instance is created and can't
    // be changed afterwards, so there is no need to protect them with a
    // separate mutex.
    checkpoints         []chaincfg.Checkpoint
    checkpointsByHeight map[int32]*chaincfg.Checkpoint
    db                  database.DB
    chainParams         *chaincfg.Params
    timeSource          MedianTimeSource
    sigCache            *txscript.SigCache
    indexManager        IndexManager
    hashCache           *txscript.HashCache

    // The following fields are calculated based upon the provided chain
    // parameters.  They are also set when the instance is created and
    // can't be changed afterwards, so there is no need to protect them with
    // a separate mutex.
    minRetargetTimespan int64 // target timespan / adjustment factor
    maxRetargetTimespan int64 // target timespan * adjustment factor
    blocksPerRetarget   int32 // target timespan / target time per block

    // chainLock protects concurrent access to the vast majority of the
    // fields in this struct below this point.
    chainLock sync.RWMutex

    // These fields are related to the memory block index.  They both have
    // their own locks, however they are often also protected by the chain
    // lock to help prevent logic races when blocks are being processed.
    //
    // index houses the entire block index in memory.  The block index is
    // a tree-shaped structure.
    //
    // bestChain tracks the current active chain by making use of an
    // efficient chain view into the block index.
    index     *blockIndex
    bestChain *chainView

    // These fields are related to handling of orphan blocks.  They are
    // protected by a combination of the chain lock and the orphan lock.
    orphanLock   sync.RWMutex
    orphans      map[chainhash.Hash]*orphanBlock
    prevOrphans  map[chainhash.Hash][]*orphanBlock
    oldestOrphan *orphanBlock

    // These fields are related to checkpoint handling.  They are protected
    // by the chain lock.
    nextCheckpoint *chaincfg.Checkpoint
    checkpointNode *blockNode

    // The state is used as a fairly efficient way to cache information
    // about the current best chain state that is returned to callers when
    // requested.  It operates on the principle of MVCC such that any time a
    // new block becomes the best block, the state pointer is replaced with
    // a new struct and the old state is left untouched.  In this way,
    // multiple callers can be pointing to different best chain states.
    // This is acceptable for most callers because the state is only being
    // queried at a specific point in time.
    //
    // In addition, some of the fields are stored in the database so the
    // chain state can be quickly reconstructed on load.
    stateLock     sync.RWMutex
    stateSnapshot *BestState

    // The following caches are used to efficiently keep track of the
    // current deployment threshold state of each rule change deployment.
    //
    // This information is stored in the database so it can be quickly
    // reconstructed on load.
    //
    // warningCaches caches the current deployment threshold state for blocks
    // in each of the **possible** deployments.  This is used in order to
    // detect when new unrecognized rule changes are being voted on and/or
    // have been activated such as will be the case when older versions of
    // the software are being used
    //
    // deploymentCaches caches the current deployment threshold state for
    // blocks in each of the actively defined deployments.
    warningCaches    []thresholdStateCache
    deploymentCaches []thresholdStateCache

    // The following fields are used to determine if certain warnings have
    // already been shown.
    //
    // unknownRulesWarned refers to warnings due to unknown rules being
    // activated.
    //
    // unknownVersionsWarned refers to warnings due to unknown versions
    // being mined.
    unknownRulesWarned    bool
    unknownVersionsWarned bool

    // The notifications field stores a slice of callbacks to be executed on
    // certain blockchain events.
    notificationsLock sync.RWMutex
    notifications     []NotificationCallback
}
```

### 1.1.2. blockNode

```go
// blockNode represents a block within the block chain and is primarily used to
// aid in selecting the best chain to be the main chain.  The main chain is
// stored into the block database.
type blockNode struct {
    // NOTE: Additions, deletions, or modifications to the order of the
    // definitions in this struct should not be changed without considering
    // how it affects alignment on 64-bit platforms.  The current order is
    // specifically crafted to result in minimal padding.  There will be
    // hundreds of thousands of these in memory, so a few extra bytes of
    // padding adds up.

    // parent is the parent block for this node.
    parent *blockNode

    // hash is the double sha 256 of the block.
    hash chainhash.Hash

    // workSum is the total amount of work in the chain up to and including
    // this node.
    workSum *big.Int

    // height is the position in the block chain.
    height int32

    // Some fields from block headers to aid in best chain selection and
    // reconstructing headers from memory.  These must be treated as
    // immutable and are intentionally ordered to avoid padding on 64-bit
    // platforms.
    version    int32
    bits       uint32
    nonce      uint32
    timestamp  int64
    merkleRoot chainhash.Hash

    // status is a bitfield representing the validation state of the block. The
    // status field, unlike the other fields, may be written to and so should
    // only be accessed using the concurrent-safe NodeStatus method on
    // blockIndex once the node has been added to the global index.
    status blockStatus
}
```

## 1.2. blockchain创建流程

>系统在启用时，在server.newServer()时会创建区块链对象，并做一些初始化工作。这里我用代码片段换掉代码，方便分析。

>**chain.go/New**

```go
// New returns a BlockChain instance using the provided configuration details.
func New(config *Config) (*BlockChain, error) {
    // Enforce required config fields.
    if config.DB == nil {
        return nil, AssertError("blockchain.New database is nil")
    }
    if config.ChainParams == nil {
        return nil, AssertError("blockchain.New chain parameters nil")
    }
    if config.TimeSource == nil {
        return nil, AssertError("blockchain.New timesource is nil")
    }

    {code_segment_1}

    {code_segment_2}

    {code_segment_3}

    {code_segment_4}

    {code_segment_5}

    {code_segment_6}

    bestNode := b.bestChain.Tip()
    log.Infof("Chain state (height %d, hash %v, totaltx %d, work %v)",
        bestNode.height, bestNode.hash, b.stateSnapshot.TotalTxns,
        bestNode.workSum)

    return &b, nil
}
```

### 1.2.1. Checkpoint索引

>**code_segment_1**

```go
// Generate a checkpoint by height map from the provided checkpoints
// and assert the provided checkpoints are sorted by height as required.
var checkpointsByHeight map[int32]*chaincfg.Checkpoint
var prevCheckpointHeight int32
if len(config.Checkpoints) > 0 {
    checkpointsByHeight = make(map[int32]*chaincfg.Checkpoint)
    for i := range config.Checkpoints {
        checkpoint := &config.Checkpoints[i]
        if checkpoint.Height <= prevCheckpointHeight {
            return nil, AssertError("blockchain.New " +
                "checkpoints are not sorted by height")
        }

        checkpointsByHeight[checkpoint.Height] = checkpoint
        prevCheckpointHeight = checkpoint.Height
    }
}
```

这段代码建立了一个hash索引，方便后面处理。 

>主网Checkpoints配置(chaincfg/params.go)如下。Checkpoint为检查点。这些区块是已经在主网生成了的。

```go
// Checkpoints ordered from oldest to newest.
Checkpoints: []Checkpoint{
    {11111, newHashFromStr("0000000069e244f73d78e8fd29ba2fd2ed618bd6fa2ee92559f542fdb26e7c1d")},
    {33333, newHashFromStr("000000002dd5588a74784eaa7ab0507a18ad16a236e7b1ce69f00d7ddfb5d0a6")},
    {74000, newHashFromStr("0000000000573993a3c9e41ce34471c079dcf5f52a0e824a81e7f953b8661a20")},
    {105000, newHashFromStr("00000000000291ce28027faea320c8d2b054b2e0fe44a773f3eefb151d6bdc97")},
    {134444, newHashFromStr("00000000000005b12ffd4cd315cd34ffd4a594f430ac814c91184a0d42d2b0fe")},
    {168000, newHashFromStr("000000000000099e61ea72015e79632f216fe6cb33d7899acb35b75c8303b763")},
    {193000, newHashFromStr("000000000000059f452a5f7340de6682a977387c17010ff6e6c3bd83ca8b1317")},
    {210000, newHashFromStr("000000000000048b95347e83192f69cf0366076336c639f9b7228e9ba171342e")},
    {216116, newHashFromStr("00000000000001b4f4b433e81ee46494af945cf96014816a4e2370f11b23df4e")},
    {225430, newHashFromStr("00000000000001c108384350f74090433e7fcf79a606b8e797f065b130575932")},
    {250000, newHashFromStr("000000000000003887df1f29024b06fc2200b55f8af8f35453d7be294df2d214")},
    {267300, newHashFromStr("000000000000000a83fbd660e918f218bf37edd92b748ad940483c7c116179ac")},
    {279000, newHashFromStr("0000000000000001ae8c72a0b0c301f67e3afca10e819efa9041e458e9bd7e40")},
    {300255, newHashFromStr("0000000000000000162804527c6e9b9f0563a280525f9d08c12041def0a0f3b2")},
    {319400, newHashFromStr("000000000000000021c6052e9becade189495d1c539aa37c58917305fd15f13b")},
    {343185, newHashFromStr("0000000000000000072b8bf361d01a6ba7d445dd024203fafc78768ed4368554")},
    {352940, newHashFromStr("000000000000000010755df42dba556bb72be6a32f3ce0b6941ce4430152c9ff")},
    {382320, newHashFromStr("00000000000000000a8dc6ed5b133d0eb2fd6af56203e4159789b092defd8ab2")},
},
```

### 1.2.2. New BlockChain

>**code_segment_2**

```go
params := config.ChainParams
targetTimespan := int64(params.TargetTimespan / time.Second)
targetTimePerBlock := int64(params.TargetTimePerBlock / time.Second)
adjustmentFactor := params.RetargetAdjustmentFactor
b := BlockChain{
    checkpoints:         config.Checkpoints,
    checkpointsByHeight: checkpointsByHeight,
    db:                  config.DB,
    chainParams:         params,
    timeSource:          config.TimeSource,
    sigCache:            config.SigCache,
    indexManager:        config.IndexManager,
    minRetargetTimespan: targetTimespan / adjustmentFactor,
    maxRetargetTimespan: targetTimespan * adjustmentFactor,
    blocksPerRetarget:   int32(targetTimespan / targetTimePerBlock),
    index:               newBlockIndex(config.DB, params),
    hashCache:           config.HashCache,
    bestChain:           newChainView(nil),
    orphans:             make(map[chainhash.Hash]*orphanBlock),
    prevOrphans:         make(map[chainhash.Hash][]*orphanBlock),
    warningCaches:       newThresholdCaches(vbNumBits),
    deploymentCaches:    newThresholdCaches(chaincfg.DefinedDeployments),
}
```

>**newBlockIndex**

```go
// newBlockIndex returns a new empty instance of a block index.  The index will
// be dynamically populated as block nodes are loaded from the database and
// manually added.
func newBlockIndex(db database.DB, chainParams *chaincfg.Params) *blockIndex {
    return &blockIndex{
        db:          db,
        chainParams: chainParams,
        index:       make(map[chainhash.Hash]*blockNode),
        dirty:       make(map[*blockNode]struct{}),
    }
}
```

### 1.2.3. initChainState

>**code_segment_3**

```go
// Initialize the chain state from the passed database.  When the db
// does not yet contain any chain state, both it and the chain state
// will be initialized to contain only the genesis block.
if err := b.initChainState(); err != nil {
    return nil, err
}
```

>**initChainState**

```go
// initChainState attempts to load and initialize the chain state from the
// database.  When the db does not yet contain any chain state, both it and the
// chain state are initialized to the genesis block.
func (b *BlockChain) initChainState() error {
    // Determine the state of the chain database. We may need to initialize
    // everything from scratch or upgrade certain buckets.
    var initialized, hasBlockIndex bool
    err := b.db.View(func(dbTx database.Tx) error {
        initialized = dbTx.Metadata().Get(chainStateKeyName) != nil
        hasBlockIndex = dbTx.Metadata().Bucket(blockIndexBucketName) != nil
        return nil
    })
    if err != nil {
        return err
    }

    if !initialized {
        // At this point the database has not already been initialized, so
        // initialize both it and the chain state to the genesis block.
        return b.createChainState()
    }

    if !hasBlockIndex {
        err := migrateBlockIndex(b.db)
        if err != nil {
            return nil
        }
    }

    {code_load_the_chain_state}
    
    // As we might have updated the index after it was loaded, we'll
    // attempt to flush the index to the DB. This will only result in a
    // write if the elements are dirty, so it'll usually be a noop.
    return b.index.flushToDB()
}
```

在initChainState中做了如下工作：

>1. 如果initialized=false, 创建chainState
>2. 否则，初始化相关数据
>3. 版本更新导致的数据结构变动要处理

#### 1.2.3.1. createChainState

```go
// createChainState initializes both the database and the chain state to the
// genesis block.  This includes creating the necessary buckets and inserting
// the genesis block, so it must only be called on an uninitialized database.
func (b *BlockChain) createChainState() error {
    // Create a new node from the genesis block and set it as the best node.
    genesisBlock := btcutil.NewBlock(b.chainParams.GenesisBlock)
    genesisBlock.SetHeight(0)
    header := &genesisBlock.MsgBlock().Header
    node := newBlockNode(header, nil)
    node.status = statusDataStored | statusValid
    b.bestChain.SetTip(node)

    // Add the new node to the index which is used for faster lookups.
    b.index.addNode(node)

    // Initialize the state related to the best block.  Since it is the
    // genesis block, use its timestamp for the median time.
    numTxns := uint64(len(genesisBlock.MsgBlock().Transactions))
    blockSize := uint64(genesisBlock.MsgBlock().SerializeSize())
    blockWeight := uint64(GetBlockWeight(genesisBlock))
    b.stateSnapshot = newBestState(node, blockSize, blockWeight, numTxns,
        numTxns, time.Unix(node.timestamp, 0))

    // Create the initial the database chain state including creating the
    // necessary index buckets and inserting the genesis block.
    err := b.db.Update(func(dbTx database.Tx) error {
        meta := dbTx.Metadata()

        // Create the bucket that houses the block index data.
        _, err := meta.CreateBucket(blockIndexBucketName)
        if err != nil {
            return err
        }

        // Create the bucket that houses the chain block hash to height
        // index.
        _, err = meta.CreateBucket(hashIndexBucketName)
        if err != nil {
            return err
        }

        // Create the bucket that houses the chain block height to hash
        // index.
        _, err = meta.CreateBucket(heightIndexBucketName)
        if err != nil {
            return err
        }

        // Create the bucket that houses the spend journal data and
        // store its version.
        _, err = meta.CreateBucket(spendJournalBucketName)
        if err != nil {
            return err
        }
        err = dbPutVersion(dbTx, utxoSetVersionKeyName,
            latestUtxoSetBucketVersion)
        if err != nil {
            return err
        }

        // Create the bucket that houses the utxo set and store its
        // version.  Note that the genesis block coinbase transaction is
        // intentionally not inserted here since it is not spendable by
        // consensus rules.
        _, err = meta.CreateBucket(utxoSetBucketName)
        if err != nil {
            return err
        }
        err = dbPutVersion(dbTx, spendJournalVersionKeyName,
            latestSpendJournalBucketVersion)
        if err != nil {
            return err
        }

        // Save the genesis block to the block index database.
        err = dbStoreBlockNode(dbTx, node)
        if err != nil {
            return err
        }

        // Add the genesis block hash to height and height to hash
        // mappings to the index.
        err = dbPutBlockIndex(dbTx, &node.hash, node.height)
        if err != nil {
            return err
        }

        // Store the current best chain state into the database.
        err = dbPutBestState(dbTx, b.stateSnapshot, node.workSum)
        if err != nil {
            return err
        }

        // Store the genesis block into the database.
        return dbStoreBlock(dbTx, genesisBlock)
    })
    return err
}
```

这个方法里，会生成创世区块。然后得到一些基本状态，包括numTxns，blockSize，blockWeight等。最后会生成一些索引的bucket等，最后保存区块到ffldb中。根据上面的代码，其实就可以看出保存一个区块有那些方面。

1. 保存block头及状态到索引
```go
// blockIndexBucketName is the name of the db bucket used to house to the
// block headers and contextual information.
blockIndexBucketName = []byte("blockheaderidx")
```
2. 添加区块索引(2个)  
```go
// heightIndexBucketName is the name of the db bucket used to house to 
// the block height -> block hash index.
heightIndexBucketName = []byte("heightidx")

// hashIndexBucketName is the name of the db bucket used to house to the
// block hash -> block height index.
hashIndexBucketName = []byte("hashidx")
```
3. 更新区块链状态
```go
// dbPutBestState uses an existing database transaction to update the best chain
// state with the given parameters.
func dbPutBestState(dbTx database.Tx, snapshot *BestState, workSum *big.Int) error {
    // Serialize the current best chain state.
    serializedData := serializeBestChainState(bestChainState{
        hash:      snapshot.Hash,
        height:    uint32(snapshot.Height),
        totalTxns: snapshot.TotalTxns,
        workSum:   workSum,
    })

    // Store the current best chain state into the database.
    return dbTx.Metadata().Put(chainStateKeyName, serializedData)
}
```

1. 保存block

保存区块时，会先保存到本地文件中，然后把路径保存到数据库中blockIdxBucket，

```go
// dbStoreBlock stores the provided block in the database if it is not already
// there. The full block data is written to ffldb.
func dbStoreBlock(dbTx database.Tx, block *btcutil.Block) error {
    hasBlock, err := dbTx.HasBlock(block.Hash())
    if err != nil {
        return err
    }
    if hasBlock {
        return nil
    }
    return dbTx.StoreBlock(block)
}
```

>**database/ffldb/db.go**

提交时处理逻辑：

```go
location, err := tx.db.store.writeBlock(blockData.bytes)
if err != nil {
    rollback()
    return err
}

// Add a record in the block index for the block.  The record
// includes the location information needed to locate the block
// on the filesystem as well as the block header since they are
// so commonly needed.
blockRow := serializeBlockLoc(location)
err = tx.blockIdxBucket.Put(blockData.hash[:], blockRow)
```

这章先不管ffldb具体逻辑，知道大致结构就行了。

#### 1.2.3.2. 加载链状态

>**code_load_the_chain_state**

下面的代码主要是加载必要的数据到内存中。

```go
// Attempt to load the chain state from the database.
err = b.db.View(func(dbTx database.Tx) error {
    // Fetch the stored chain state from the database metadata.
    // When it doesn't exist, it means the database hasn't been
    // initialized for use with chain yet, so break out now to allow
    // that to happen under a writable database transaction.
    serializedData := dbTx.Metadata().Get(chainStateKeyName)
    log.Tracef("Serialized chain state: %x", serializedData)
    state, err := deserializeBestChainState(serializedData)
    if err != nil {
        return err
    }

    // Load all of the headers from the data for the known best
    // chain and construct the block index accordingly.  Since the
    // number of nodes are already known, perform a single alloc
    // for them versus a whole bunch of little ones to reduce
    // pressure on the GC.
    log.Infof("Loading block index...")

    blockIndexBucket := dbTx.Metadata().Bucket(blockIndexBucketName)

    // Determine how many blocks will be loaded into the index so we can
    // allocate the right amount.
    var blockCount int32
    cursor := blockIndexBucket.Cursor()
    for ok := cursor.First(); ok; ok = cursor.Next() {
        blockCount++
    }
    blockNodes := make([]blockNode, blockCount)

    var i int32
    var lastNode *blockNode
    cursor = blockIndexBucket.Cursor()
    for ok := cursor.First(); ok; ok = cursor.Next() {
        header, status, err := deserializeBlockRow(cursor.Value())
        if err != nil {
            return err
        }

        // Determine the parent block node. Since we iterate block headers
        // in order of height, if the blocks are mostly linear there is a
        // very good chance the previous header processed is the parent.
        var parent *blockNode
        if lastNode == nil {
            blockHash := header.BlockHash()
            if !blockHash.IsEqual(b.chainParams.GenesisHash) {
                return AssertError(fmt.Sprintf("initChainState: Expected "+
                    "first entry in block index to be genesis block, "+
                    "found %s", blockHash))
            }
        } else if header.PrevBlock == lastNode.hash {
            // Since we iterate block headers in order of height, if the
            // blocks are mostly linear there is a very good chance the
            // previous header processed is the parent.
            parent = lastNode
        } else {
            parent = b.index.LookupNode(&header.PrevBlock)
            if parent == nil {
                return AssertError(fmt.Sprintf("initChainState: Could "+
                    "not find parent for block %s", header.BlockHash()))
            }
        }

        // Initialize the block node for the block, connect it,
        // and add it to the block index.
        node := &blockNodes[i]
        initBlockNode(node, header, parent)
        node.status = status
        b.index.addNode(node)

        lastNode = node
        i++
    }

    // Set the best chain view to the stored best state.
    tip := b.index.LookupNode(&state.hash)
    if tip == nil {
        return AssertError(fmt.Sprintf("initChainState: cannot find "+
            "chain tip %s in block index", state.hash))
    }
    b.bestChain.SetTip(tip)

    // Load the raw block bytes for the best block.
    blockBytes, err := dbTx.FetchBlock(&state.hash)
    if err != nil {
        return err
    }
    var block wire.MsgBlock
    err = block.Deserialize(bytes.NewReader(blockBytes))
    if err != nil {
        return err
    }

    // As a final consistency check, we'll run through all the
    // nodes which are ancestors of the current chain tip, and mark
    // them as valid if they aren't already marked as such.  This
    // is a safe assumption as all the block before the current tip
    // are valid by definition.
    for iterNode := tip; iterNode != nil; iterNode = iterNode.parent {
        // If this isn't already marked as valid in the index, then
        // we'll mark it as valid now to ensure consistency once
        // we're up and running.
        if !iterNode.status.KnownValid() {
            log.Infof("Block %v (height=%v) ancestor of "+
                "chain tip not marked as valid, "+
                "upgrading to valid for consistency",
                iterNode.hash, iterNode.height)

            b.index.SetStatusFlags(iterNode, statusValid)
        }
    }

    // Initialize the state related to the best block.
    blockSize := uint64(len(blockBytes))
    blockWeight := uint64(GetBlockWeight(btcutil.NewBlock(&block)))
    numTxns := uint64(len(block.Transactions))
    b.stateSnapshot = newBestState(tip, blockSize, blockWeight,
        numTxns, state.totalTxns, tip.CalcPastMedianTime())

    return nil
})
if err != nil {
    return err
}

```

1. 先从chainStateKeyName中加载BestChainState，这个对象维护了当前主链尾基本状态
2. 把全部区块头信息加载到内存，生成BlockNode链表，并且维护到index中
3. 根据state中的hash从index中找到主链尾节点blocknode,并且把这个区块加载出来
4. 把主链上所有节点的状态设置为statusValid，防止系统关闭前异常导致的状态更新失败影响。
5. 更新blcokcahin.stateSnapshot

### 1.2.4. UpgradeDbBuckets

>**code_segment_4:**

```go
// Perform any upgrades to the various chain-specific buckets as needed.
if err := b.maybeUpgradeDbBuckets(config.Interrupt); err != nil {
    return nil, err
}
```

版本更新处理。看下这个方法的说明：  

```text
maybeUpgradeDbBuckets checks the database version of the buckets used by this
package and performs any needed upgrades to bring them to the latest version.

All buckets used by this package are guaranteed to be the latest version if
this function returns without error.
```

### 1.2.5. IndexManager初始化

>**code_segment_5:**

```go
// Initialize and catch up all of the currently active optional indexes
// as needed.
if config.IndexManager != nil {
    err := config.IndexManager.Init(&b, config.Interrupt)
    if err != nil {
        return nil, err
    }
}
```

```go
// Init initializes the enabled indexes.  This is called during chain
// initialization and primarily consists of catching up all indexes to the
// current best chain tip.  This is necessary since each index can be disabled
// and re-enabled at any time and attempting to catch-up indexes at the same
// time new blocks are being downloaded would lead to an overall longer time to
// catch up due to the I/O contention.
//
// This is part of the blockchain.IndexManager interface.
func (m *Manager) Init(chain *blockchain.BlockChain, interrupt <-chan struct{}) error {
    // Nothing to do when no indexes are enabled.
    if len(m.enabledIndexes) == 0 {
        return nil
    }

    if interruptRequested(interrupt) {
        return errInterruptRequested
    }

    // Finish and drops that were previously interrupted.
    if err := m.maybeFinishDrops(interrupt); err != nil {
        return err
    }

    // Create the initial state for the indexes as needed.
    err := m.db.Update(func(dbTx database.Tx) error {
        // Create the bucket for the current tips as needed.
        meta := dbTx.Metadata()
        _, err := meta.CreateBucketIfNotExists(indexTipsBucketName)
        if err != nil {
            return err
        }

        return m.maybeCreateIndexes(dbTx)
    })
    if err != nil {
        return err
    }

    // Initialize each of the enabled indexes.
    for _, indexer := range m.enabledIndexes {
        if err := indexer.Init(); err != nil {
            return err
        }
    }

    // Rollback indexes to the main chain if their tip is an orphaned fork.
    // This is fairly unlikely, but it can happen if the chain is
    // reorganized while the index is disabled.  This has to be done in
    // reverse order because later indexes can depend on earlier ones.
    for i := len(m.enabledIndexes); i > 0; i-- {
        indexer := m.enabledIndexes[i-1]

        // Fetch the current tip for the index.
        var height int32
        var hash *chainhash.Hash
        err := m.db.View(func(dbTx database.Tx) error {
            idxKey := indexer.Key()
            hash, height, err = dbFetchIndexerTip(dbTx, idxKey)
            return err
        })
        if err != nil {
            return err
        }

        // Nothing to do if the index does not have any entries yet.
        if height == -1 {
            continue
        }

        // Loop until the tip is a block that exists in the main chain.
        initialHeight := height
        for !chain.MainChainHasBlock(hash) {
            // At this point the index tip is orphaned, so load the
            // orphaned block from the database directly and
            // disconnect it from the index.  The block has to be
            // loaded directly since it is no longer in the main
            // chain and thus the chain.BlockByHash function would
            // error.
            var block *btcutil.Block
            err := m.db.View(func(dbTx database.Tx) error {
                blockBytes, err := dbTx.FetchBlock(hash)
                if err != nil {
                    return err
                }
                block, err = btcutil.NewBlockFromBytes(blockBytes)
                if err != nil {
                    return err
                }
                block.SetHeight(height)
                return err
            })
            if err != nil {
                return err
            }

            // We'll also grab the set of outputs spent by this
            // block so we can remove them from the index.
            spentTxos, err := chain.FetchSpendJournal(block)
            if err != nil {
                return err
            }

            // With the block and stxo set for that block retrieved,
            // we can now update the index itself.
            err = m.db.Update(func(dbTx database.Tx) error {
                // Remove all of the index entries associated
                // with the block and update the indexer tip.
                err = dbIndexDisconnectBlock(
                    dbTx, indexer, block, spentTxos,
                )
                if err != nil {
                    return err
                }

                // Update the tip to the previous block.
                hash = &block.MsgBlock().Header.PrevBlock
                height--

                return nil
            })
            if err != nil {
                return err
            }

            if interruptRequested(interrupt) {
                return errInterruptRequested
            }
        }

        if initialHeight != height {
            log.Infof("Removed %d orphaned blocks from %s "+
                "(heights %d to %d)", initialHeight-height,
                indexer.Name(), height+1, initialHeight)
        }
    }

    // Fetch the current tip heights for each index along with tracking the
    // lowest one so the catchup code only needs to start at the earliest
    // block and is able to skip connecting the block for the indexes that
    // don't need it.
    bestHeight := chain.BestSnapshot().Height
    lowestHeight := bestHeight
    indexerHeights := make([]int32, len(m.enabledIndexes))
    err = m.db.View(func(dbTx database.Tx) error {
        for i, indexer := range m.enabledIndexes {
            idxKey := indexer.Key()
            hash, height, err := dbFetchIndexerTip(dbTx, idxKey)
            if err != nil {
                return err
            }

            log.Debugf("Current %s tip (height %d, hash %v)",
                indexer.Name(), height, hash)
            indexerHeights[i] = height
            if height < lowestHeight {
                lowestHeight = height
            }
        }
        return nil
    })
    if err != nil {
        return err
    }

    // Nothing to index if all of the indexes are caught up.
    if lowestHeight == bestHeight {
        return nil
    }

    // Create a progress logger for the indexing process below.
    progressLogger := newBlockProgressLogger("Indexed", log)

    // At this point, one or more indexes are behind the current best chain
    // tip and need to be caught up, so log the details and loop through
    // each block that needs to be indexed.
    log.Infof("Catching up indexes from height %d to %d", lowestHeight,
        bestHeight)
    for height := lowestHeight + 1; height <= bestHeight; height++ {
        // Load the block for the height since it is required to index
        // it.
        block, err := chain.BlockByHeight(height)
        if err != nil {
            return err
        }

        if interruptRequested(interrupt) {
            return errInterruptRequested
        }

        // Connect the block for all indexes that need it.
        var spentTxos []blockchain.SpentTxOut
        for i, indexer := range m.enabledIndexes {
            // Skip indexes that don't need to be updated with this
            // block.
            if indexerHeights[i] >= height {
                continue
            }

            // When the index requires all of the referenced txouts
            // and they haven't been loaded yet, they need to be
            // retrieved from the spend journal.
            if spentTxos == nil && indexNeedsInputs(indexer) {
                spentTxos, err = chain.FetchSpendJournal(block)
                if err != nil {
                    return err
                }
            }

            err := m.db.Update(func(dbTx database.Tx) error {
                return dbIndexConnectBlock(
                    dbTx, indexer, block, spentTxos,
                )
            })
            if err != nil {
                return err
            }
            indexerHeights[i] = height
        }

        // Log indexing progress.
        progressLogger.LogBlockHeight(block)

        if interruptRequested(interrupt) {
            return errInterruptRequested
        }
    }

    log.Infof("Indexes caught up to height %d", bestHeight)
    return nil
}
```

系统设计了三个索引服务，但是默认不会都开启。

#### 1.2.5.1. transaction index

```text

// -----------------------------------------------------------------------------
// The transaction index consists of an entry for every transaction in the main
// chain.  In order to significantly optimize the space requirements a separate
// index which provides an internal mapping between each block that has been
// indexed and a unique ID for use within the hash to location mappings.  The ID
// is simply a sequentially incremented uint32.  This is useful because it is
// only 4 bytes versus 32 bytes hashes and thus saves a ton of space in the
// index.
//
// There are three buckets used in total.  The first bucket maps the hash of
// each transaction to the specific block location.  The second bucket maps the
// hash of each block to the unique ID and the third maps that ID back to the
// block hash.
//
// NOTE: Although it is technically possible for multiple transactions to have
// the same hash as long as the previous transaction with the same hash is fully
// spent, this code only stores the most recent one because doing otherwise
// would add a non-trivial amount of space and overhead for something that will
// realistically never happen per the probability and even if it did, the old
// one must be fully spent and so the most likely transaction a caller would
// want for a given hash is the most recent one anyways.
//
// The serialized format for keys and values in the block hash to ID bucket is:
//   <hash> = <ID>
//
//   Field           Type              Size
//   hash            chainhash.Hash    32 bytes
//   ID              uint32            4 bytes
//   -----
//   Total: 36 bytes
//
// The serialized format for keys and values in the ID to block hash bucket is:
//   <ID> = <hash>
//
//   Field           Type              Size
//   ID              uint32            4 bytes
//   hash            chainhash.Hash    32 bytes
//   -----
//   Total: 36 bytes
//
// The serialized format for the keys and values in the tx index bucket is:
//
//   <txhash> = <block id><start offset><tx length>
//
//   Field           Type              Size
//   txhash          chainhash.Hash    32 bytes
//   block id        uint32            4 bytes
//   start offset    uint32          4 bytes
//   tx length       uint32          4 bytes
//   -----
//   Total: 44 bytes
// -----------------------------------------------------------------------------
```

```go
// TxIndex implements a transaction by hash index.  That is to say, it supports
// querying all transactions by their hash.
type TxIndex struct {
    db         database.DB
    curBlockID uint32
}
```

#### 1.2.5.2. address index

```text

// -----------------------------------------------------------------------------
// The address index maps addresses referenced in the blockchain to a list of
// all the transactions involving that address.  Transactions are stored
// according to their order of appearance in the blockchain.  That is to say
// first by block height and then by offset inside the block.  It is also
// important to note that this implementation requires the transaction index
// since it is needed in order to catch up old blocks due to the fact the spent
// outputs will already be pruned from the utxo set.
//
// The approach used to store the index is similar to a log-structured merge
// tree (LSM tree) and is thus similar to how leveldb works internally.
//
// Every address consists of one or more entries identified by a level starting
// from 0 where each level holds a maximum number of entries such that each
// subsequent level holds double the maximum of the previous one.  In equation
// form, the number of entries each level holds is 2^n * firstLevelMaxSize.
//
// New transactions are appended to level 0 until it becomes full at which point
// the entire level 0 entry is appended to the level 1 entry and level 0 is
// cleared.  This process continues until level 1 becomes full at which point it
// will be appended to level 2 and cleared and so on.
//
// The result of this is the lower levels contain newer transactions and the
// transactions within each level are ordered from oldest to newest.
//
// The intent of this approach is to provide a balance between space efficiency
// and indexing cost.  Storing one entry per transaction would have the lowest
// indexing cost, but would waste a lot of space because the same address hash
// would be duplicated for every transaction key.  On the other hand, storing a
// single entry with all transactions would be the most space efficient, but
// would cause indexing cost to grow quadratically with the number of
// transactions involving the same address.  The approach used here provides
// logarithmic insertion and retrieval.
//
// The serialized key format is:
//
//   <addr type><addr hash><level>
//
//   Field           Type      Size
//   addr type       uint8     1 byte
//   addr hash       hash160   20 bytes
//   level           uint8     1 byte
//   -----
//   Total: 22 bytes
//
// The serialized value format is:
//
//   [<block id><start offset><tx length>,...]
//
//   Field           Type      Size
//   block id        uint32    4 bytes
//   start offset    uint32    4 bytes
//   tx length       uint32    4 bytes
//   -----
//   Total: 12 bytes per indexed tx
// -----------------------------------------------------------------------------
```

```go
// AddrIndex implements a transaction by address index.  That is to say, it
// supports querying all transactions that reference a given address because
// they are either crediting or debiting the address.  The returned transactions
// are ordered according to their order of appearance in the blockchain.  In
// other words, first by block height and then by offset inside the block.
//
// In addition, support is provided for a memory-only index of unconfirmed
// transactions such as those which are kept in the memory pool before inclusion
// in a block.
type AddrIndex struct {
    // The following fields are set when the instance is created and can't
    // be changed afterwards, so there is no need to protect them with a
    // separate mutex.
    db          database.DB
    chainParams *chaincfg.Params

    // The following fields are used to quickly link transactions and
    // addresses that have not been included into a block yet when an
    // address index is being maintained.  The are protected by the
    // unconfirmedLock field.
    //
    // The txnsByAddr field is used to keep an index of all transactions
    // which either create an output to a given address or spend from a
    // previous output to it keyed by the address.
    //
    // The addrsByTx field is essentially the reverse and is used to
    // keep an index of all addresses which a given transaction involves.
    // This allows fairly efficient updates when transactions are removed
    // once they are included into a block.
    unconfirmedLock sync.RWMutex
    txnsByAddr      map[[addrKeySize]byte]map[chainhash.Hash]*btcutil.Tx
    addrsByTx       map[chainhash.Hash]map[[addrKeySize]byte]struct{}
}
```

#### 1.2.5.3. committed filter index

```go
// CfIndex implements a committed filter (cf) by hash index.
type CfIndex struct {
    db          database.DB
    chainParams *chaincfg.Params
}
```

### 1.2.6. initThresholdCaches

>**code_segment_6:**



```go
// Initialize rule change threshold state caches.
if err := b.initThresholdCaches(); err != nil {
    return nil, err
}
```

```go
// initThresholdCaches initializes the threshold state caches for each warning
// bit and defined deployment and provides warnings if the chain is current per
// the warnUnknownVersions and warnUnknownRuleActivations functions.
func (b *BlockChain) initThresholdCaches() error {
    // Initialize the warning and deployment caches by calculating the
    // threshold state for each of them.  This will ensure the caches are
    // populated and any states that needed to be recalculated due to
    // definition changes is done now.
    prevNode := b.bestChain.Tip().parent
    for bit := uint32(0); bit < vbNumBits; bit++ {
        checker := bitConditionChecker{bit: bit, chain: b}
        cache := &b.warningCaches[bit]
        _, err := b.thresholdState(prevNode, checker, cache)
        if err != nil {
            return err
        }
    }
    for id := 0; id < len(b.chainParams.Deployments); id++ {
        deployment := &b.chainParams.Deployments[id]
        cache := &b.deploymentCaches[id]
        checker := deploymentChecker{deployment: deployment, chain: b}
        _, err := b.thresholdState(prevNode, checker, cache)
        if err != nil {
            return err
        }
    }

    // No warnings about unknown rules or versions until the chain is
    // current.
    if b.isCurrent() {
        // Warn if a high enough percentage of the last blocks have
        // unexpected versions.
        bestNode := b.bestChain.Tip()
        if err := b.warnUnknownVersions(bestNode); err != nil {
            return err
        }

        // Warn if any unknown new rules are either about to activate or
        // have already been activated.
        if err := b.warnUnknownRuleActivations(bestNode); err != nil {
            return err
        }
    }

    return nil
}

```

#### 1.2.6.1. isCurrent

判断当前节点是否已经同步最新区块。在主链尾节点调度小于最后一个检查点之前，可以明确知道是没有同步到最新区块的。超过检查点之后，只能根据AdjustedTime时间判断。由于AdjustedTime是根据连接成功的peer中时间生成的调整时间，因此，返回true并不能一定说明节点同步到最新状态，但是也相关不多。

```go
func (b *BlockChain) isCurrent() bool {
    // Not current if the latest main (best) chain height is before the
    // latest known good checkpoint (when checkpoints are enabled).
    checkpoint := b.LatestCheckpoint()
    if checkpoint != nil && b.bestChain.Tip().height < checkpoint.Height {
        return false
    }

    // Not current if the latest best block has a timestamp before 24 hours
    // ago.
    //
    // The chain appears to be current if none of the checks reported
    // otherwise.
    minus24Hours := b.timeSource.AdjustedTime().Add(-24 * time.Hour).Unix()
    return b.bestChain.Tip().timestamp >= minus24Hours
}
```