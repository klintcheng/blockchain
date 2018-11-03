# 1. 挖矿

<!-- TOC -->

- [1. 挖矿](#1-%E6%8C%96%E7%9F%BF)
    - [1.1. 简介](#11-%E7%AE%80%E4%BB%8B)
    - [1.2. 内存池](#12-%E5%86%85%E5%AD%98%E6%B1%A0)
        - [1.2.1. 基本结构](#121-%E5%9F%BA%E6%9C%AC%E7%BB%93%E6%9E%84)
        - [1.2.2. 交易来源](#122-%E4%BA%A4%E6%98%93%E6%9D%A5%E6%BA%90)
            - [1.2.2.1. ProcessTransaction](#1221-processtransaction)
                - [1.2.2.1.1. maybeAcceptTransaction](#12211-maybeaccepttransaction)
                - [1.2.2.1.2. addTransaction](#12212-addtransaction)
    - [1.3. miner](#13-miner)
        - [1.3.1. CPUMiner](#131-cpuminer)
        - [1.3.2. miningWorkerController](#132-miningworkercontroller)
        - [1.3.3. generateBlocks](#133-generateblocks)
            - [1.3.3.1. solveBlock](#1331-solveblock)
                - [1.3.3.1.1. UpdateExtraNonce](#13311-updateextranonce)

<!-- /TOC -->

## 1.1. 简介

作为分布式系统最重要的一环，挖矿就是一个竞争成为提案者(proposer)的过程。在中心化系统中常用的paxos和raft等算法无法解决节点不可信问题。bitcoin采用POW工作量证明机制来证明区块是否合法。当一个挖矿节点生成一个新节点，并广播出去，当另一个挖矿节点收到这个区块并验证通过之后，会停止之前的挖矿计算，也就是之前的工作无效。开始下一轮竞争生成新的区块。

挖矿逻辑就是先要从内存池选择一定量的交易，然后不停的修改区块头中的nonce值，使生成的区块hash小于难度值。

```go
// server.go中启用miner的代码：

policy := mining.Policy{
    BlockMinWeight:    cfg.BlockMinWeight,
    BlockMaxWeight:    cfg.BlockMaxWeight,
    BlockMinSize:      cfg.BlockMinSize,
    BlockMaxSize:      cfg.BlockMaxSize,
    BlockPrioritySize: cfg.BlockPrioritySize,
    TxMinFreeFee:      cfg.minRelayTxFee,
}
blockTemplateGenerator := mining.NewBlkTmplGenerator(&policy,
    s.chainParams, s.txMemPool, s.chain, s.timeSource,
    s.sigCache, s.hashCache)
s.cpuMiner = cpuminer.New(&cpuminer.Config{
    ChainParams:            chainParams,
    BlockTemplateGenerator: blockTemplateGenerator,
    MiningAddrs:            cfg.miningAddrs,
    ProcessBlock:           s.syncManager.ProcessBlock,
    ConnectedCount:         s.ConnectedCount,
    IsCurrent:              s.syncManager.IsCurrent,
})

// Start the CPU miner if generation is enabled.
if cfg.Generate {
    s.cpuMiner.Start()
}
```

- policy：挖矿限制条件。
- blockTemplateGenerator：区块生成模板。

## 1.2. 内存池

挖矿会从内存池中选择交易，我们先看看内存池的结构和交易数据的来源。

>创建mempool对象

```go
txC := mempool.Config{
    Policy: mempool.Policy{
        DisableRelayPriority: cfg.NoRelayPriority,
        AcceptNonStd:         cfg.RelayNonStd,
        FreeTxRelayLimit:     cfg.FreeTxRelayLimit,
        MaxOrphanTxs:         cfg.MaxOrphanTxs,
        MaxOrphanTxSize:      defaultMaxOrphanTxSize,
        MaxSigOpCostPerTx:    blockchain.MaxBlockSigOpsCost / 4,
        MinRelayTxFee:        cfg.minRelayTxFee,
        MaxTxVersion:         2,
    },
    ChainParams:    chainParams,
    FetchUtxoView:  s.chain.FetchUtxoView,
    BestHeight:     func() int32 { return s.chain.BestSnapshot().Height },
    MedianTimePast: func() time.Time { return s.chain.BestSnapshot().MedianTime },
    CalcSequenceLock: func(tx *btcutil.Tx, view *blockchain.UtxoViewpoint) (*blockchain.SequenceLock, error) {
        return s.chain.CalcSequenceLock(tx, view, true)
    },
    IsDeploymentActive: s.chain.IsDeploymentActive,
    SigCache:           s.sigCache,
    HashCache:          s.hashCache,
    AddrIndex:          s.addrIndex,
    FeeEstimator:       s.feeEstimator,
}
s.txMemPool = mempool.New(&txC)
```

### 1.2.1. 基本结构

```go
// TxPool is used as a source of transactions that need to be mined into blocks
// and relayed to other peers.  It is safe for concurrent access from multiple
// peers.
type TxPool struct {
    // The following variables must only be used atomically.
    lastUpdated int64 // last time pool was updated

    mtx           sync.RWMutex
    cfg           Config
    pool          map[chainhash.Hash]*TxDesc
    orphans       map[chainhash.Hash]*orphanTx
    orphansByPrev map[wire.OutPoint]map[chainhash.Hash]*btcutil.Tx
    outpoints     map[wire.OutPoint]*btcutil.Tx
    pennyTotal    float64 // exponentially decaying total for penny spends.
    lastPennyUnix int64   // unix time of last ``penny spend''

    // nextExpireScan is the time after which the orphan pool will be
    // scanned in order to evict orphans.  This is NOT a hard deadline as
    // the scan will only run when an orphan is added to the pool as opposed
    // to on an unconditional timer.
    nextExpireScan time.Time
}

// TxDesc is a descriptor containing a transaction in the mempool along with
// additional metadata.
type TxDesc struct {
    mining.TxDesc

    // StartingPriority is the priority of the transaction when it was added
    // to the pool.
    StartingPriority float64
}

// mining.TxDesc:

// TxDesc is a descriptor about a transaction in a transaction source along with
// additional metadata.
type TxDesc struct {
    // Tx is the transaction associated with the entry.
    Tx *btcutil.Tx

    // Added is the time when the entry was added to the source pool.
    Added time.Time

    // Height is the block height when the entry was added to the the source
    // pool.
    Height int32

    // Fee is the total fee the transaction associated with the entry pays.
    Fee int64

    // FeePerKB is the fee the transaction pays in Satoshi per 1000 bytes.
    FeePerKB int64
}

// orphanTx is normal transaction that references an ancestor transaction
// that is not yet available.  It also contains additional information related
// to it such as an expiration time to help prevent caching the orphan forever.
type orphanTx struct {
    tx         *btcutil.Tx
    tag        Tag
    expiration time.Time
}
```



### 1.2.2. 交易来源

在之前章节介绍过，交易的传播会在SyncManager中处理，也就是说一个节点要么收到rpc发过来的交易信息，要么是收到其它节点传播过来的交易，都会验证之后添加到内存池。

> handleTxMsg处理交易消息

```go
// handleTxMsg handles transaction messages from all peers.
func (sm *SyncManager) handleTxMsg(tmsg *txMsg) {
    peer := tmsg.peer
    state, exists := sm.peerStates[peer]
    if !exists {
        log.Warnf("Received tx message from unknown peer %s", peer)
        return
    }

    // NOTE:  BitcoinJ, and possibly other wallets, don't follow the spec of
    // sending an inventory message and allowing the remote peer to decide
    // whether or not they want to request the transaction via a getdata
    // message.  Unfortunately, the reference implementation permits
    // unrequested data, so it has allowed wallets that don't follow the
    // spec to proliferate.  While this is not ideal, there is no check here
    // to disconnect peers for sending unsolicited transactions to provide
    // interoperability.
    txHash := tmsg.tx.Hash()

    // Ignore transactions that we have already rejected.  Do not
    // send a reject message here because if the transaction was already
    // rejected, the transaction was unsolicited.
    if _, exists = sm.rejectedTxns[*txHash]; exists {
        log.Debugf("Ignoring unsolicited previously rejected "+
            "transaction %v from %s", txHash, peer)
        return
    }

    // Process the transaction to include validation, insertion in the
    // memory pool, orphan handling, etc.
    acceptedTxs, err := sm.txMemPool.ProcessTransaction(tmsg.tx,
        true, true, mempool.Tag(peer.ID()))

    // Remove transaction from request maps. Either the mempool/chain
    // already knows about it and as such we shouldn't have any more
    // instances of trying to fetch it, or we failed to insert and thus
    // we'll retry next time we get an inv.
    delete(state.requestedTxns, *txHash)
    delete(sm.requestedTxns, *txHash)

    if err != nil {
        // Do not request this transaction again until a new block
        // has been processed.
        sm.rejectedTxns[*txHash] = struct{}{}
        sm.limitMap(sm.rejectedTxns, maxRejectedTxns)

        // When the error is a rule error, it means the transaction was
        // simply rejected as opposed to something actually going wrong,
        // so log it as such.  Otherwise, something really did go wrong,
        // so log it as an actual error.
        if _, ok := err.(mempool.RuleError); ok {
            log.Debugf("Rejected transaction %v from %s: %v",
                txHash, peer, err)
        } else {
            log.Errorf("Failed to process transaction %v: %v",
                txHash, err)
        }

        // Convert the error into an appropriate reject message and
        // send it.
        code, reason := mempool.ErrToRejectErr(err)
        peer.PushRejectMsg(wire.CmdTx, code, reason, txHash, false)
        return
    }

    sm.peerNotifier.AnnounceNewTransactions(acceptedTxs)
}
```

>**我们主要看下核心逻辑，也就是内存池的处理ProcessTransaction**

#### 1.2.2.1. ProcessTransaction

```go

// ProcessTransaction is the main workhorse for handling insertion of new
// free-standing transactions into the memory pool.  It includes functionality
// such as rejecting duplicate transactions, ensuring transactions follow all
// rules, orphan transaction handling, and insertion into the memory pool.
//
// It returns a slice of transactions added to the mempool.  When the
// error is nil, the list will include the passed transaction itself along
// with any additional orphan transaactions that were added as a result of
// the passed one being accepted.
//
// This function is safe for concurrent access.
func (mp *TxPool) ProcessTransaction(tx *btcutil.Tx, allowOrphan, rateLimit bool, tag Tag) ([]*TxDesc, error) {
    log.Tracef("Processing transaction %v", tx.Hash())

    // Protect concurrent access.
    mp.mtx.Lock()
    defer mp.mtx.Unlock()

    // Potentially accept the transaction to the memory pool.
    missingParents, txD, err := mp.maybeAcceptTransaction(tx, true, rateLimit,
        true)
    if err != nil {
        return nil, err
    }

    if len(missingParents) == 0 {
        // Accept any orphan transactions that depend on this
        // transaction (they may no longer be orphans if all inputs
        // are now available) and repeat for those accepted
        // transactions until there are no more.
        newTxs := mp.processOrphans(tx)
        acceptedTxs := make([]*TxDesc, len(newTxs)+1)

        // Add the parent transaction first so remote nodes
        // do not add orphans.
        acceptedTxs[0] = txD
        copy(acceptedTxs[1:], newTxs)

        return acceptedTxs, nil
    }

    // The transaction is an orphan (has inputs missing).  Reject
    // it if the flag to allow orphans is not set.
    if !allowOrphan {
        // Only use the first missing parent transaction in
        // the error message.
        //
        // NOTE: RejectDuplicate is really not an accurate
        // reject code here, but it matches the reference
        // implementation and there isn't a better choice due
        // to the limited number of reject codes.  Missing
        // inputs is assumed to mean they are already spent
        // which is not really always the case.
        str := fmt.Sprintf("orphan transaction %v references "+
            "outputs of unknown or fully-spent "+
            "transaction %v", tx.Hash(), missingParents[0])
        return nil, txRuleError(wire.RejectDuplicate, str)
    }

    // Potentially add the orphan transaction to the orphan pool.
    err = mp.maybeAddOrphan(tx, tag)
    return nil, err
}
```

上面的函数中，只做两件事:

1. 接受交易，并处理依赖这个交易的孤儿交易。
2. allowOrphan=true，添加到到孤儿池。

##### 1.2.2.1.1. maybeAcceptTransaction

这个方法基本上包括了一个交易所有的验证逻辑。验证的代码很多，但是每一个验证前都有原因说明。

```go
// maybeAcceptTransaction is the internal function which implements the public
// MaybeAcceptTransaction.  See the comment for MaybeAcceptTransaction for
// more details.
//
// This function MUST be called with the mempool lock held (for writes).
func (mp *TxPool) maybeAcceptTransaction(tx *btcutil.Tx, isNew, rateLimit, rejectDupOrphans bool) ([]*chainhash.Hash, *TxDesc, error) {
    txHash := tx.Hash()

    // If a transaction has iwtness data, and segwit isn't active yet, If
    // segwit isn't active yet, then we won't accept it into the mempool as
    // it can't be mined yet.
    if tx.MsgTx().HasWitness() {
        segwitActive, err := mp.cfg.IsDeploymentActive(chaincfg.DeploymentSegwit)
        if err != nil {
            return nil, nil, err
        }

        if !segwitActive {
            str := fmt.Sprintf("transaction %v has witness data, "+
                "but segwit isn't active yet", txHash)
            return nil, nil, txRuleError(wire.RejectNonstandard, str)
        }
    }

    // Don't accept the transaction if it already exists in the pool.  This
    // applies to orphan transactions as well when the reject duplicate
    // orphans flag is set.  This check is intended to be a quick check to
    // weed out duplicates.
    if mp.isTransactionInPool(txHash) || (rejectDupOrphans &&
        mp.isOrphanInPool(txHash)) {

        str := fmt.Sprintf("already have transaction %v", txHash)
        return nil, nil, txRuleError(wire.RejectDuplicate, str)
    }

    // Perform preliminary sanity checks on the transaction.  This makes
    // use of blockchain which contains the invariant rules for what
    // transactions are allowed into blocks.
    err := blockchain.CheckTransactionSanity(tx)
    if err != nil {
        if cerr, ok := err.(blockchain.RuleError); ok {
            return nil, nil, chainRuleError(cerr)
        }
        return nil, nil, err
    }

    // A standalone transaction must not be a coinbase transaction.
    if blockchain.IsCoinBase(tx) {
        str := fmt.Sprintf("transaction %v is an individual coinbase",
            txHash)
        return nil, nil, txRuleError(wire.RejectInvalid, str)
    }

    // Get the current height of the main chain.  A standalone transaction
    // will be mined into the next block at best, so its height is at least
    // one more than the current height.
    bestHeight := mp.cfg.BestHeight()
    nextBlockHeight := bestHeight + 1

    medianTimePast := mp.cfg.MedianTimePast()

    // Don't allow non-standard transactions if the network parameters
    // forbid their acceptance.
    if !mp.cfg.Policy.AcceptNonStd {
        err = checkTransactionStandard(tx, nextBlockHeight,
            medianTimePast, mp.cfg.Policy.MinRelayTxFee,
            mp.cfg.Policy.MaxTxVersion)
        if err != nil {
            // Attempt to extract a reject code from the error so
            // it can be retained.  When not possible, fall back to
            // a non standard error.
            rejectCode, found := extractRejectCode(err)
            if !found {
                rejectCode = wire.RejectNonstandard
            }
            str := fmt.Sprintf("transaction %v is not standard: %v",
                txHash, err)
            return nil, nil, txRuleError(rejectCode, str)
        }
    }

    // The transaction may not use any of the same outputs as other
    // transactions already in the pool as that would ultimately result in a
    // double spend.  This check is intended to be quick and therefore only
    // detects double spends within the transaction pool itself.  The
    // transaction could still be double spending coins from the main chain
    // at this point.  There is a more in-depth check that happens later
    // after fetching the referenced transaction inputs from the main chain
    // which examines the actual spend data and prevents double spends.
    err = mp.checkPoolDoubleSpend(tx)
    if err != nil {
        return nil, nil, err
    }

    // Fetch all of the unspent transaction outputs referenced by the inputs
    // to this transaction.  This function also attempts to fetch the
    // transaction itself to be used for detecting a duplicate transaction
    // without needing to do a separate lookup.
    utxoView, err := mp.fetchInputUtxos(tx)
    if err != nil {
        if cerr, ok := err.(blockchain.RuleError); ok {
            return nil, nil, chainRuleError(cerr)
        }
        return nil, nil, err
    }

    // Don't allow the transaction if it exists in the main chain and is not
    // not already fully spent.
    prevOut := wire.OutPoint{Hash: *txHash}
    for txOutIdx := range tx.MsgTx().TxOut {
        prevOut.Index = uint32(txOutIdx)
        entry := utxoView.LookupEntry(prevOut)
        if entry != nil && !entry.IsSpent() {
            return nil, nil, txRuleError(wire.RejectDuplicate,
                "transaction already exists")
        }
        utxoView.RemoveEntry(prevOut)
    }

    // Transaction is an orphan if any of the referenced transaction outputs
    // don't exist or are already spent.  Adding orphans to the orphan pool
    // is not handled by this function, and the caller should use
    // maybeAddOrphan if this behavior is desired.
    var missingParents []*chainhash.Hash
    for outpoint, entry := range utxoView.Entries() {
        if entry == nil || entry.IsSpent() {
            // Must make a copy of the hash here since the iterator
            // is replaced and taking its address directly would
            // result in all of the entries pointing to the same
            // memory location and thus all be the final hash.
            hashCopy := outpoint.Hash
            missingParents = append(missingParents, &hashCopy)
        }
    }
    if len(missingParents) > 0 {
        return missingParents, nil, nil
    }

    // Don't allow the transaction into the mempool unless its sequence
    // lock is active, meaning that it'll be allowed into the next block
    // with respect to its defined relative lock times.
    sequenceLock, err := mp.cfg.CalcSequenceLock(tx, utxoView)
    if err != nil {
        if cerr, ok := err.(blockchain.RuleError); ok {
            return nil, nil, chainRuleError(cerr)
        }
        return nil, nil, err
    }
    if !blockchain.SequenceLockActive(sequenceLock, nextBlockHeight,
        medianTimePast) {
        return nil, nil, txRuleError(wire.RejectNonstandard,
            "transaction's sequence locks on inputs not met")
    }

    // Perform several checks on the transaction inputs using the invariant
    // rules in blockchain for what transactions are allowed into blocks.
    // Also returns the fees associated with the transaction which will be
    // used later.
    txFee, err := blockchain.CheckTransactionInputs(tx, nextBlockHeight,
        utxoView, mp.cfg.ChainParams)
    if err != nil {
        if cerr, ok := err.(blockchain.RuleError); ok {
            return nil, nil, chainRuleError(cerr)
        }
        return nil, nil, err
    }

    // Don't allow transactions with non-standard inputs if the network
    // parameters forbid their acceptance.
    if !mp.cfg.Policy.AcceptNonStd {
        err := checkInputsStandard(tx, utxoView)
        if err != nil {
            // Attempt to extract a reject code from the error so
            // it can be retained.  When not possible, fall back to
            // a non standard error.
            rejectCode, found := extractRejectCode(err)
            if !found {
                rejectCode = wire.RejectNonstandard
            }
            str := fmt.Sprintf("transaction %v has a non-standard "+
                "input: %v", txHash, err)
            return nil, nil, txRuleError(rejectCode, str)
        }
    }

    // NOTE: if you modify this code to accept non-standard transactions,
    // you should add code here to check that the transaction does a
    // reasonable number of ECDSA signature verifications.

    // Don't allow transactions with an excessive number of signature
    // operations which would result in making it impossible to mine.  Since
    // the coinbase address itself can contain signature operations, the
    // maximum allowed signature operations per transaction is less than
    // the maximum allowed signature operations per block.
    // TODO(roasbeef): last bool should be conditional on segwit activation
    sigOpCost, err := blockchain.GetSigOpCost(tx, false, utxoView, true, true)
    if err != nil {
        if cerr, ok := err.(blockchain.RuleError); ok {
            return nil, nil, chainRuleError(cerr)
        }
        return nil, nil, err
    }
    if sigOpCost > mp.cfg.Policy.MaxSigOpCostPerTx {
        str := fmt.Sprintf("transaction %v sigop cost is too high: %d > %d",
            txHash, sigOpCost, mp.cfg.Policy.MaxSigOpCostPerTx)
        return nil, nil, txRuleError(wire.RejectNonstandard, str)
    }

    // Don't allow transactions with fees too low to get into a mined block.
    //
    // Most miners allow a free transaction area in blocks they mine to go
    // alongside the area used for high-priority transactions as well as
    // transactions with fees.  A transaction size of up to 1000 bytes is
    // considered safe to go into this section.  Further, the minimum fee
    // calculated below on its own would encourage several small
    // transactions to avoid fees rather than one single larger transaction
    // which is more desirable.  Therefore, as long as the size of the
    // transaction does not exceeed 1000 less than the reserved space for
    // high-priority transactions, don't require a fee for it.
    serializedSize := GetTxVirtualSize(tx)
    minFee := calcMinRequiredTxRelayFee(serializedSize,
        mp.cfg.Policy.MinRelayTxFee)
    if serializedSize >= (DefaultBlockPrioritySize-1000) && txFee < minFee {
        str := fmt.Sprintf("transaction %v has %d fees which is under "+
            "the required amount of %d", txHash, txFee,
            minFee)
        return nil, nil, txRuleError(wire.RejectInsufficientFee, str)
    }

    // Require that free transactions have sufficient priority to be mined
    // in the next block.  Transactions which are being added back to the
    // memory pool from blocks that have been disconnected during a reorg
    // are exempted.
    if isNew && !mp.cfg.Policy.DisableRelayPriority && txFee < minFee {
        currentPriority := mining.CalcPriority(tx.MsgTx(), utxoView,
            nextBlockHeight)
        if currentPriority <= mining.MinHighPriority {
            str := fmt.Sprintf("transaction %v has insufficient "+
                "priority (%g <= %g)", txHash,
                currentPriority, mining.MinHighPriority)
            return nil, nil, txRuleError(wire.RejectInsufficientFee, str)
        }
    }

    // Free-to-relay transactions are rate limited here to prevent
    // penny-flooding with tiny transactions as a form of attack.
    if rateLimit && txFee < minFee {
        nowUnix := time.Now().Unix()
        // Decay passed data with an exponentially decaying ~10 minute
        // window - matches bitcoind handling.
        mp.pennyTotal *= math.Pow(1.0-1.0/600.0,
            float64(nowUnix-mp.lastPennyUnix))
        mp.lastPennyUnix = nowUnix

        // Are we still over the limit?
        if mp.pennyTotal >= mp.cfg.Policy.FreeTxRelayLimit*10*1000 {
            str := fmt.Sprintf("transaction %v has been rejected "+
                "by the rate limiter due to low fees", txHash)
            return nil, nil, txRuleError(wire.RejectInsufficientFee, str)
        }
        oldTotal := mp.pennyTotal

        mp.pennyTotal += float64(serializedSize)
        log.Tracef("rate limit: curTotal %v, nextTotal: %v, "+
            "limit %v", oldTotal, mp.pennyTotal,
            mp.cfg.Policy.FreeTxRelayLimit*10*1000)
    }

    // Verify crypto signatures for each input and reject the transaction if
    // any don't verify.
    err = blockchain.ValidateTransactionScripts(tx, utxoView,
        txscript.StandardVerifyFlags, mp.cfg.SigCache,
        mp.cfg.HashCache)
    if err != nil {
        if cerr, ok := err.(blockchain.RuleError); ok {
            return nil, nil, chainRuleError(cerr)
        }
        return nil, nil, err
    }

    // Add to transaction pool.
    txD := mp.addTransaction(utxoView, tx, bestHeight, txFee)

    log.Debugf("Accepted transaction %v (pool size: %v)", txHash,
        len(mp.pool))

    return nil, txD, nil
}
```

##### 1.2.2.1.2. addTransaction

```go
// addTransaction adds the passed transaction to the memory pool.  It should
// not be called directly as it doesn't perform any validation.  This is a
// helper for maybeAcceptTransaction.
//
// This function MUST be called with the mempool lock held (for writes).
func (mp *TxPool) addTransaction(utxoView *blockchain.UtxoViewpoint, tx *btcutil.Tx, height int32, fee int64) *TxDesc {
    // Add the transaction to the pool and mark the referenced outpoints
    // as spent by the pool.
    txD := &TxDesc{
        TxDesc: mining.TxDesc{
            Tx:       tx,
            Added:    time.Now(),
            Height:   height,
            Fee:      fee,
            FeePerKB: fee * 1000 / GetTxVirtualSize(tx),
        },
        StartingPriority: mining.CalcPriority(tx.MsgTx(), utxoView, height),
    }

    mp.pool[*tx.Hash()] = txD
    for _, txIn := range tx.MsgTx().TxIn {
        mp.outpoints[txIn.PreviousOutPoint] = tx
    }
    atomic.StoreInt64(&mp.lastUpdated, time.Now().Unix())

    // Add unconfirmed address index entries associated with the transaction
    // if enabled.
    if mp.cfg.AddrIndex != nil {
        mp.cfg.AddrIndex.AddUnconfirmedTx(tx, utxoView)
    }

    // Record this tx for fee estimation if enabled.
    if mp.cfg.FeeEstimator != nil {
        mp.cfg.FeeEstimator.ObserveTransaction(txD)
    }

    return txD
}
```

这个方法清晰的说明了内存池中数据是如果存放的。同时，也看到了AddrIndex中保存了未确认的交易。最后会记录交易到FeeEstimator中，FeeEstimator用于作为区块模板打包交易时评估。

## 1.3. miner

### 1.3.1. CPUMiner

```go
// Config is a descriptor containing the cpu miner configuration.
type Config struct {
    // ChainParams identifies which chain parameters the cpu miner is
    // associated with.
    ChainParams *chaincfg.Params

    // BlockTemplateGenerator identifies the instance to use in order to
    // generate block templates that the miner will attempt to solve.
    BlockTemplateGenerator *mining.BlkTmplGenerator

    // MiningAddrs is a list of payment addresses to use for the generated
    // blocks.  Each generated block will randomly choose one of them.
    MiningAddrs []btcutil.Address

    // ProcessBlock defines the function to call with any solved blocks.
    // It typically must run the provided block through the same set of
    // rules and handling as any other block coming from the network.
    ProcessBlock func(*btcutil.Block, blockchain.BehaviorFlags) (bool, error)

    // ConnectedCount defines the function to use to obtain how many other
    // peers the server is connected to.  This is used by the automatic
    // persistent mining routine to determine whether or it should attempt
    // mining.  This is useful because there is no point in mining when not
    // connected to any peers since there would no be anyone to send any
    // found blocks to.
    ConnectedCount func() int32

    // IsCurrent defines the function to use to obtain whether or not the
    // block chain is current.  This is used by the automatic persistent
    // mining routine to determine whether or it should attempt mining.
    // This is useful because there is no point in mining if the chain is
    // not current since any solved blocks would be on a side chain and and
    // up orphaned anyways.
    IsCurrent func() bool
}

// CPUMiner provides facilities for solving blocks (mining) using the CPU in
// a concurrency-safe manner.  It consists of two main goroutines -- a speed
// monitor and a controller for worker goroutines which generate and solve
// blocks.  The number of goroutines can be set via the SetMaxGoRoutines
// function, but the default is based on the number of processor cores in the
// system which is typically sufficient.
type CPUMiner struct {
    sync.Mutex
    g                 *mining.BlkTmplGenerator
    cfg               Config
    numWorkers        uint32
    started           bool
    discreteMining    bool
    submitBlockLock   sync.Mutex
    wg                sync.WaitGroup
    workerWg          sync.WaitGroup
    updateNumWorkers  chan struct{}
    queryHashesPerSec chan float64
    updateHashes      chan uint64
    speedMonitorQuit  chan struct{}
    quit              chan struct{}
}

// Start begins the CPU mining process as well as the speed monitor used to
// track hashing metrics.  Calling this function when the CPU miner has
// already been started will have no effect.
//
// This function is safe for concurrent access.
func (m *CPUMiner) Start() {
    m.Lock()
    defer m.Unlock()

    // Nothing to do if the miner is already running or if running in
    // discrete mode (using GenerateNBlocks).
    if m.started || m.discreteMining {
        return
    }

    m.quit = make(chan struct{})
    m.speedMonitorQuit = make(chan struct{})
    m.wg.Add(2)
    go m.speedMonitor()
    go m.miningWorkerController()

    m.started = true
    log.Infof("CPU miner started")
}
```

上面列了cpuminer基本结构，同时，启动时做了那些事。speedMonitor先放一放，我们先看下miningWorkerController做的事。

### 1.3.2. miningWorkerController

```go
// miningWorkerController launches the worker goroutines that are used to
// generate block templates and solve them.  It also provides the ability to
// dynamically adjust the number of running worker goroutines.
//
// It must be run as a goroutine.
func (m *CPUMiner) miningWorkerController() {
    // launchWorkers groups common code to launch a specified number of
    // workers for generating blocks.
    var runningWorkers []chan struct{}
    launchWorkers := func(numWorkers uint32) {
        for i := uint32(0); i < numWorkers; i++ {
            quit := make(chan struct{})
            runningWorkers = append(runningWorkers, quit)

            m.workerWg.Add(1)
            go m.generateBlocks(quit)
        }
    }

    // Launch the current number of workers by default.
    runningWorkers = make([]chan struct{}, 0, m.numWorkers)
    launchWorkers(m.numWorkers)

out:
    for {
        select {
        // Update the number of running workers.
        case <-m.updateNumWorkers:
            // No change.
            numRunning := uint32(len(runningWorkers))
            if m.numWorkers == numRunning {
                continue
            }

            // Add new workers.
            if m.numWorkers > numRunning {
                launchWorkers(m.numWorkers - numRunning)
                continue
            }

            // Signal the most recently created goroutines to exit.
            for i := numRunning - 1; i >= m.numWorkers; i-- {
                close(runningWorkers[i])
                runningWorkers[i] = nil
                runningWorkers = runningWorkers[:i]
            }

        case <-m.quit:
            for _, quit := range runningWorkers {
                close(quit)
            }
            break out
        }
    }

    // Wait until all workers shut down to stop the speed monitor since
    // they rely on being able to send updates to it.
    m.workerWg.Wait()
    close(m.speedMonitorQuit)
    m.wg.Done()
}
```

这里首先创建了默认的runtime.NumCPU()个工作线程，然后启用select,监听updateNumWorkers，动态添加或者删除某个worker。

### 1.3.3. generateBlocks

这个方法包括了创建一个区块的主要逻辑。

generateBlocks is a worker that is controlled by the miningWorkerController. It is self contained in that it creates block templates and attempts to solve them while detecting when it is performing stale work and reacting accordingly by generating a new block template.  When a block is solved, it is submitted.

```go
// It must be run as a goroutine.
func (m *CPUMiner) generateBlocks(quit chan struct{}) {
    log.Tracef("Starting generate blocks worker")

    // Start a ticker which is used to signal checks for stale work and
    // updates to the speed monitor.
    ticker := time.NewTicker(time.Second * hashUpdateSecs)
    defer ticker.Stop()
out:
    for {
        // Quit when the miner is stopped.
        select {
        case <-quit:
            break out
        default:
            // Non-blocking select to fall through
        }

        // Wait until there is a connection to at least one other peer
        // since there is no way to relay a found block or receive
        // transactions to work on when there are no connected peers.
        if m.cfg.ConnectedCount() == 0 {
            time.Sleep(time.Second)
            continue
        }

        // No point in searching for a solution before the chain is
        // synced.  Also, grab the same lock as used for block
        // submission, since the current block will be changing and
        // this would otherwise end up building a new block template on
        // a block that is in the process of becoming stale.
        m.submitBlockLock.Lock()
        curHeight := m.g.BestSnapshot().Height
        if curHeight != 0 && !m.cfg.IsCurrent() {
            m.submitBlockLock.Unlock()
            time.Sleep(time.Second)
            continue
        }

        // Choose a payment address at random.
        rand.Seed(time.Now().UnixNano())
        payToAddr := m.cfg.MiningAddrs[rand.Intn(len(m.cfg.MiningAddrs))]

        // Create a new block template using the available transactions
        // in the memory pool as a source of transactions to potentially
        // include in the block.
        template, err := m.g.NewBlockTemplate(payToAddr)
        m.submitBlockLock.Unlock()
        if err != nil {
            errStr := fmt.Sprintf("Failed to create new block "+
                "template: %v", err)
            log.Errorf(errStr)
            continue
        }

        // Attempt to solve the block.  The function will exit early
        // with false when conditions that trigger a stale block, so
        // a new block template can be generated.  When the return is
        // true a solution was found, so submit the solved block.
        if m.solveBlock(template.Block, curHeight+1, ticker, quit) {
            block := btcutil.NewBlock(template.Block)
            m.submitBlock(block)
        }
    }

    m.workerWg.Done()
    log.Tracef("Generate blocks worker done")
}
```

首先在for循环中会用非阻塞的方法读取quit，然后如下处理：

1. 当前节点是连接数，如果为0就要等待。
2. 当前主链是否已经达到最新，如果没达到，挖矿是无意义的。
3. 选择一个支付地址，如果你竞争成功，生成的区块被接受，在coinbase里会包括你的收益。
4. 根据payToAddr生成一个区块模板，这个区块模板中会包括出mempool中取出的交易。
5. 最重要的一环，解决算法难题，找到合法的nonce值使区块生效
6. 提前生成的区块，广播出去，使其它节点得到这个区块。

其中，NewBlockTemplate是最复杂的逻辑，我们放到最后看。

#### 1.3.3.1. solveBlock

solveBlock attempts to find some combination of a nonce, extra nonce, and current timestamp which makes the passed block hash to a value less than the target difficulty.  The timestamp is updated periodically and the passed block is modified with all tweaks during this process.  This means that when the function returns true, the block is ready for submission.

```go
// This function will return early with false when conditions that trigger a
// stale block such as a new block showing up or periodically when there are
// new transactions and enough time has elapsed without finding a solution.
func (m *CPUMiner) solveBlock(msgBlock *wire.MsgBlock, blockHeight int32,
    ticker *time.Ticker, quit chan struct{}) bool {

    // Choose a random extra nonce offset for this block template and
    // worker.
    enOffset, err := wire.RandomUint64()
    if err != nil {
        log.Errorf("Unexpected error while generating random "+
            "extra nonce offset: %v", err)
        enOffset = 0
    }

    // Create some convenience variables.
    header := &msgBlock.Header
    targetDifficulty := blockchain.CompactToBig(header.Bits)

    // Initial state.
    lastGenerated := time.Now()
    lastTxUpdate := m.g.TxSource().LastUpdated()
    hashesCompleted := uint64(0)

    // Note that the entire extra nonce range is iterated and the offset is
    // added relying on the fact that overflow will wrap around 0 as
    // provided by the Go spec.
    for extraNonce := uint64(0); extraNonce < maxExtraNonce; extraNonce++ {
        // Update the extra nonce in the block template with the
        // new value by regenerating the coinbase script and
        // setting the merkle root to the new value.
        m.g.UpdateExtraNonce(msgBlock, blockHeight, extraNonce+enOffset)

        // Search through the entire nonce range for a solution while
        // periodically checking for early quit and stale block
        // conditions along with updates to the speed monitor.
        for i := uint32(0); i <= maxNonce; i++ {
            select {
            case <-quit:
                return false

            case <-ticker.C:
                m.updateHashes <- hashesCompleted
                hashesCompleted = 0

                // The current block is stale if the best block
                // has changed.
                best := m.g.BestSnapshot()
                if !header.PrevBlock.IsEqual(&best.Hash) {
                    return false
                }

                // The current block is stale if the memory pool
                // has been updated since the block template was
                // generated and it has been at least one
                // minute.
                if lastTxUpdate != m.g.TxSource().LastUpdated() &&
                    time.Now().After(lastGenerated.Add(time.Minute)) {

                    return false
                }

                m.g.UpdateBlockTime(msgBlock)

            default:
                // Non-blocking select to fall through
            }

            // Update the nonce and hash the block header.  Each
            // hash is actually a double sha256 (two hashes), so
            // increment the number of hashes completed for each
            // attempt accordingly.
            header.Nonce = i
            hash := header.BlockHash()
            hashesCompleted += 2

            // The block is solved when the new block hash is less
            // than the target difficulty.  Yay!
            if blockchain.HashToBig(&hash).Cmp(targetDifficulty) <= 0 {
                m.updateHashes <- hashesCompleted
                return true
            }
        }
    }

    return false
}
```

从这个方法中可以看出来，影响一个区块hash的有两个值：

1. extraNonce，这个值是在coinbase中，会导致header中的MerkleRoot变动
2. header.Nonce

由于在挖矿的过程中，可能会收到新的合法区块，模板中的交易会变动，因此在开始前会周期性检查。最后的代码就是比较hash与目标难度。

>**blockchain.HashToBig(&hash).Cmp(targetDifficulty) <= 0**

##### 1.3.3.1.1. UpdateExtraNonce

```go
// UpdateExtraNonce updates the extra nonce in the coinbase script of the passed
// block by regenerating the coinbase script with the passed value and block
// height.  It also recalculates and updates the new merkle root that results
// from changing the coinbase script.
func (g *BlkTmplGenerator) UpdateExtraNonce(msgBlock *wire.MsgBlock, blockHeight int32, extraNonce uint64) error {
    coinbaseScript, err := standardCoinbaseScript(blockHeight, extraNonce)
    if err != nil {
        return err
    }
    if len(coinbaseScript) > blockchain.MaxCoinbaseScriptLen {
        return fmt.Errorf("coinbase transaction script length "+
            "of %d is out of range (min: %d, max: %d)",
            len(coinbaseScript), blockchain.MinCoinbaseScriptLen,
            blockchain.MaxCoinbaseScriptLen)
    }
    msgBlock.Transactions[0].TxIn[0].SignatureScript = coinbaseScript

    // TODO(davec): A btcutil.Block should use saved in the state to avoid
    // recalculating all of the other transaction hashes.
    // block.Transactions[0].InvalidateCache()

    // Recalculate the merkle root with the updated extra nonce.
    block := btcutil.NewBlock(msgBlock)
    merkles := blockchain.BuildMerkleTreeStore(block.Transactions(), false)
    msgBlock.Header.MerkleRoot = *merkles[len(merkles)-1]
    return nil
}

func standardCoinbaseScript(nextBlockHeight int32, extraNonce uint64) ([]byte, error) {
    return txscript.NewScriptBuilder().AddInt64(int64(nextBlockHeight)).
        AddInt64(int64(extraNonce)).AddData([]byte(CoinbaseFlags)).
        Script()
}
```

