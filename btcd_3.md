# 1. blockchain
<!-- TOC -->

- [1. blockchain](#1-blockchain)
    - [1.1. 创建blockchain](#11-创建blockchain)

<!-- /TOC -->
## 1.1. 创建blockchain

>系统在启用时，在server.newServer()调用了创建区块链对象。

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

    // Initialize the chain state from the passed database.  When the db
    // does not yet contain any chain state, both it and the chain state
    // will be initialized to contain only the genesis block.
    if err := b.initChainState(); err != nil {
        return nil, err
    }

    // Perform any upgrades to the various chain-specific buckets as needed.
    if err := b.maybeUpgradeDbBuckets(config.Interrupt); err != nil {
        return nil, err
    }

    // Initialize and catch up all of the currently active optional indexes
    // as needed.
    if config.IndexManager != nil {
        err := config.IndexManager.Init(&b, config.Interrupt)
        if err != nil {
            return nil, err
        }
    }

    // Initialize rule change threshold state caches.
    if err := b.initThresholdCaches(); err != nil {
        return nil, err
    }

    bestNode := b.bestChain.Tip()
    log.Infof("Chain state (height %d, hash %v, totaltx %d, work %v)",
        bestNode.height, bestNode.hash, b.stateSnapshot.TotalTxns,
        bestNode.workSum)

    return &b, nil
}
```

1. new BlockChain
2. initChainState

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

    // As we might have updated the index after it was loaded, we'll
    // attempt to flush the index to the DB. This will only result in a
    // write if the elements are dirty, so it'll usually be a noop.
    return b.index.flushToDB()
}
```