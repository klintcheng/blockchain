# 1. 区块处理

<!-- TOC -->

- [1. 区块处理](#1-区块处理)
    - [1.1. 介绍](#11-介绍)
        - [1.1.1. ProcessBlock](#111-processblock)
        - [1.1.2. 参数详解](#112-参数详解)
    - [1.2. 内部逻辑](#12-内部逻辑)
        - [1.2.1. blockExists](#121-blockexists)
        - [1.2.2. checkBlockSanity](#122-checkblocksanity)
            - [1.2.2.1. checkBlockHeaderSanity](#1221-checkblockheadersanity)
            - [1.2.2.2. IsCoinBase](#1222-iscoinbase)
            - [1.2.2.3. CheckTransactionSanity](#1223-checktransactionsanity)
        - [1.2.3. findPreviousCheckpoint](#123-findpreviouscheckpoint)
            - [1.2.3.1. checkpointNode比较](#1231-checkpointnode比较)
        - [1.2.4. addOrphanBlock](#124-addorphanblock)
        - [1.2.5. maybeAcceptBlock](#125-maybeacceptblock)
            - [1.2.5.1. checkBlockContext](#1251-checkblockcontext)
            - [1.2.5.2. checkBlockHeaderContext](#1252-checkblockheadercontext)
        - [1.2.6. dbStoreBlock之block.Bytes()](#126-dbstoreblock之blockbytes)
            - [1.2.6.1. 序列化常识](#1261-序列化常识)
            - [1.2.6.2. BtcEncode](#1262-btcencode)
        - [1.2.7. newBlockNode](#127-newblocknode)
        - [1.2.8. AddNode](#128-addnode)
    - [1.3. 通用方法说明](#13-通用方法说明)
        - [1.3.1. BuildMerkleTreeStore](#131-buildmerkletreestore)
        - [1.3.2. calcNextRequiredDifficulty](#132-calcnextrequireddifficulty)
        - [1.3.3. IsFinalizedTransaction](#133-isfinalizedtransaction)
        - [1.3.4. ValidateWitnessCommitment](#134-validatewitnesscommitment)

<!-- /TOC -->

## 1.1. 介绍

在上一章，同步管理会同步区块，并且，新的区块被传播过来时，节点也要对区块做大量的处理，维护起来。本章我们从ProcessBlock开始，看看区块的处理流程。

>ProcessBlock is the main workhorse for handling insertion of new blocks into the block chain.  It includes functionality such as rejecting duplicate blocks, ensuring blocks follow all rules, orphan handling, and insertion into the block chain along with best chain selection and reorganization. When no errors occurred during processing, the first return value indicates whether or not the block is on the main chain and the second indicates whether or not the block is an orphan.

### 1.1.1. ProcessBlock

```go
func (b *BlockChain) ProcessBlock(block *btcutil.Block, flags BehaviorFlags) (bool, bool, error) {
    b.chainLock.Lock()
    defer b.chainLock.Unlock()

    fastAdd := flags&BFFastAdd == BFFastAdd

    blockHash := block.Hash()
    log.Tracef("Processing block %v", blockHash)

    // The block must not already exist in the main chain or side chains.
    exists, err := b.blockExists(blockHash)
    if err != nil {
        return false, false, err
    }
    if exists {
        str := fmt.Sprintf("already have block %v", blockHash)
        return false, false, ruleError(ErrDuplicateBlock, str)
    }

    // The block must not already exist as an orphan.
    if _, exists := b.orphans[*blockHash]; exists {
        str := fmt.Sprintf("already have block (orphan) %v", blockHash)
        return false, false, ruleError(ErrDuplicateBlock, str)
    }

    // Perform preliminary sanity checks on the block and its transactions.
    err = checkBlockSanity(block, b.chainParams.PowLimit, b.timeSource, flags)
    if err != nil {
        return false, false, err
    }

    // Find the previous checkpoint and perform some additional checks based
    // on the checkpoint.  This provides a few nice properties such as
    // preventing old side chain blocks before the last checkpoint,
    // rejecting easy to mine, but otherwise bogus, blocks that could be
    // used to eat memory, and ensuring expected (versus claimed) proof of
    // work requirements since the previous checkpoint are met.
    blockHeader := &block.MsgBlock().Header
    checkpointNode, err := b.findPreviousCheckpoint()
    if err != nil {
        return false, false, err
    }
    if checkpointNode != nil {
        // Ensure the block timestamp is after the checkpoint timestamp.
        checkpointTime := time.Unix(checkpointNode.timestamp, 0)
        if blockHeader.Timestamp.Before(checkpointTime) {
            str := fmt.Sprintf("block %v has timestamp %v before "+
                "last checkpoint timestamp %v", blockHash,
                blockHeader.Timestamp, checkpointTime)
            return false, false, ruleError(ErrCheckpointTimeTooOld, str)
        }
        if !fastAdd {
            // Even though the checks prior to now have already ensured the
            // proof of work exceeds the claimed amount, the claimed amount
            // is a field in the block header which could be forged.  This
            // check ensures the proof of work is at least the minimum
            // expected based on elapsed time since the last checkpoint and
            // maximum adjustment allowed by the retarget rules.
            duration := blockHeader.Timestamp.Sub(checkpointTime)
            requiredTarget := CompactToBig(b.calcEasiestDifficulty(
                checkpointNode.bits, duration))
            currentTarget := CompactToBig(blockHeader.Bits)
            if currentTarget.Cmp(requiredTarget) > 0 {
                str := fmt.Sprintf("block target difficulty of %064x "+
                    "is too low when compared to the previous "+
                    "checkpoint", currentTarget)
                return false, false, ruleError(ErrDifficultyTooLow, str)
            }
        }
    }

    // Handle orphan blocks.
    prevHash := &blockHeader.PrevBlock
    prevHashExists, err := b.blockExists(prevHash)
    if err != nil {
        return false, false, err
    }
    if !prevHashExists {
        log.Infof("Adding orphan block %v with parent %v", blockHash, prevHash)
        b.addOrphanBlock(block)

        return false, true, nil
    }

    // The block has passed all context independent checks and appears sane
    // enough to potentially accept it into the block chain.
    isMainChain, err := b.maybeAcceptBlock(block, flags)
    if err != nil {
        return false, false, err
    }

    // Accept any orphan blocks that depend on this block (they are
    // no longer orphans) and repeat for those accepted blocks until
    // there are no more.
    err = b.processOrphans(blockHash, flags)
    if err != nil {
        return false, false, err
    }

    log.Debugf("Accepted block %v", blockHash)

    return isMainChain, false, nil
}
```

### 1.1.2. 参数详解

>第一个参数block为要处理的block：

```go

// Block defines a bitcoin block that provides easier and more efficient
// manipulation of raw blocks.  It also memoizes hashes for the block and its
// transactions on their first access so subsequent accesses don't have to
// repeat the relatively expensive hashing operations.
type Block struct {
    msgBlock                 *wire.MsgBlock  // Underlying MsgBlock
    serializedBlock          []byte          // Serialized bytes for the block
    serializedBlockNoWitness []byte          // Serialized bytes for block w/o witness data
    blockHash                *chainhash.Hash // Cached block hash
    blockHeight              int32           // Height in the main block chain
    transactions             []*Tx           // Transactions
    txnsGenerated            bool            // ALL wrapped transactions generated
}

type Hash [HashSize]byte

type MsgBlock struct {
    Header       BlockHeader
    Transactions []*MsgTx
}

type MsgTx struct {
    Version  int32
    TxIn     []*TxIn
    TxOut    []*TxOut
    LockTime uint32
}

type TxIn struct {
    PreviousOutPoint OutPoint
    SignatureScript  []byte
    Witness          TxWitness
    Sequence         uint32
}

type OutPoint struct {
    Hash  chainhash.Hash
    Index uint32
}

// TxWitness defines the witness for a TxIn. A witness is to be interpreted as
// a slice of byte slices, or a stack with one or many elements.
type TxWitness [][]byte

type TxOut struct {
    Value    int64
    PkScript []byte
}
```

> 第二个参数为标识

在syncManager中处理区块(handleBlockMsg)时，如果headersFirstMode=true.如果区块是当前节点之前在启动同步时请求的，它就会把flags设置为blockchain.BFFastAdd。说明这个区块是验证过的。

```go
behaviorFlags := blockchain.BFNone
if sm.headersFirstMode {
    firstNodeEl := sm.headerList.Front()
    if firstNodeEl != nil {
        firstNode := firstNodeEl.Value.(*headerNode)
        if blockHash.IsEqual(firstNode.hash) {
            behaviorFlags |= blockchain.BFFastAdd
            if firstNode.hash.IsEqual(sm.nextCheckpoint.Hash) {
                isCheckpointBlock = true
            } else {
                sm.headerList.Remove(firstNodeEl)
            }
        }
    }
}
```

## 1.2. 内部逻辑

下面会列出一些重要的逻辑，其中一些明显的检查判断之类的会忽略，直接看代码很清楚。

### 1.2.1. blockExists

首先要检查这个区块是否已经在主链或者侧链上。

在blockExists这个方法体内，先检查blockIndex, blockIndex会在内存中维护一份索引，因此查询速度快。然后先在检查区块是否在blockIdxBucket中，这个过程是通过调用leveldb接口查询的。

如果查询数据库中查到了数据，但是在最后代码dbFetchHeightByHash中判断block不在主链中MainChain.也是会返回false，因为后面它可能会变成主链中的区块。

```go
func (b *BlockChain) blockExists(hash *chainhash.Hash) (bool, error) {
    // Check block index first (could be main chain or side chain blocks).
    if b.index.HaveBlock(hash) {
        return true, nil
    }

    // Check in the database.
    var exists bool
    err := b.db.View(func(dbTx database.Tx) error {
        var err error
        exists, err = dbTx.HasBlock(hash)
        if err != nil || !exists {
            return err
        }

        // Ignore side chain blocks in the database.  This is necessary
        // because there is not currently any record of the associated
        // block index data such as its block height, so it's not yet
        // possible to efficiently load the block and do anything useful
        // with it.
        //
        // Ultimately the entire block index should be serialized
        // instead of only the current main chain so it can be consulted
        // directly.
        _, err = dbFetchHeightByHash(dbTx, hash)
        if isNotInMainChainErr(err) {
            exists = false
            return nil
        }
        return err
    })
    return exists, err
}
```

### 1.2.2. checkBlockSanity

这个方法是个非常重要的方法，检查区块是否正常。

```go
// checkBlockSanity performs some preliminary checks on a block to ensure it is
// sane before continuing with block processing.  These checks are context free.
//
// The flags do not modify the behavior of this function directly, however they
// are needed to pass along to checkBlockHeaderSanity.
func checkBlockSanity(block *btcutil.Block, powLimit *big.Int, timeSource MedianTimeSource, flags BehaviorFlags) error {
    msgBlock := block.MsgBlock()
    header := &msgBlock.Header
    err := checkBlockHeaderSanity(header, powLimit, timeSource, flags)
    if err != nil {
        return err
    }

    // A block must have at least one transaction.
    numTx := len(msgBlock.Transactions)
    if numTx == 0 {
        return ruleError(ErrNoTransactions, "block does not contain "+
            "any transactions")
    }

    // A block must not have more transactions than the max block payload or
    // else it is certainly over the weight limit.
    if numTx > MaxBlockBaseSize {
        str := fmt.Sprintf("block contains too many transactions - "+
            "got %d, max %d", numTx, MaxBlockBaseSize)
        return ruleError(ErrBlockTooBig, str)
    }

    // A block must not exceed the maximum allowed block payload when
    // serialized.
    serializedSize := msgBlock.SerializeSizeStripped()
    if serializedSize > MaxBlockBaseSize {
        str := fmt.Sprintf("serialized block is too big - got %d, "+
            "max %d", serializedSize, MaxBlockBaseSize)
        return ruleError(ErrBlockTooBig, str)
    }

    // The first transaction in a block must be a coinbase.
    transactions := block.Transactions()
    if !IsCoinBase(transactions[0]) {
        return ruleError(ErrFirstTxNotCoinbase, "first transaction in "+
            "block is not a coinbase")
    }

    // A block must not have more than one coinbase.
    for i, tx := range transactions[1:] {
        if IsCoinBase(tx) {
            str := fmt.Sprintf("block contains second coinbase at "+
                "index %d", i+1)
            return ruleError(ErrMultipleCoinbases, str)
        }
    }

    // Do some preliminary checks on each transaction to ensure they are
    // sane before continuing.
    for _, tx := range transactions {
        err := CheckTransactionSanity(tx)
        if err != nil {
            return err
        }
    }

    // Build merkle tree and ensure the calculated merkle root matches the
    // entry in the block header.  This also has the effect of caching all
    // of the transaction hashes in the block to speed up future hash
    // checks.  Bitcoind builds the tree here and checks the merkle root
    // after the following checks, but there is no reason not to check the
    // merkle root matches here.
    merkles := BuildMerkleTreeStore(block.Transactions(), false)
    calculatedMerkleRoot := merkles[len(merkles)-1]
    if !header.MerkleRoot.IsEqual(calculatedMerkleRoot) {
        str := fmt.Sprintf("block merkle root is invalid - block "+
            "header indicates %v, but calculated value is %v",
            header.MerkleRoot, calculatedMerkleRoot)
        return ruleError(ErrBadMerkleRoot, str)
    }

    // Check for duplicate transactions.  This check will be fairly quick
    // since the transaction hashes are already cached due to building the
    // merkle tree above.
    existingTxHashes := make(map[chainhash.Hash]struct{})
    for _, tx := range transactions {
        hash := tx.Hash()
        if _, exists := existingTxHashes[*hash]; exists {
            str := fmt.Sprintf("block contains duplicate "+
                "transaction %v", hash)
            return ruleError(ErrDuplicateTx, str)
        }
        existingTxHashes[*hash] = struct{}{}
    }

    // The number of signature operations must be less than the maximum
    // allowed per block.
    totalSigOps := 0
    for _, tx := range transactions {
        // We could potentially overflow the accumulator so check for
        // overflow.
        lastSigOps := totalSigOps
        totalSigOps += (CountSigOps(tx) * WitnessScaleFactor)
        if totalSigOps < lastSigOps || totalSigOps > MaxBlockSigOpsCost {
            str := fmt.Sprintf("block contains too many signature "+
                "operations - got %v, max %v", totalSigOps,
                MaxBlockSigOpsCost)
            return ruleError(ErrTooManySigOps, str)
        }
    }

    return nil
}
```

其中重点的需要说明检查列举一下，在下面添加详解：

1. checkBlockHeaderSanity 检查区块头是否正常
2. IsCoinBase 检查第一个交易是否为CoinBase交易
3. CheckTransactionSanity 检查所有交易是否正常

#### 1.2.2.1. checkBlockHeaderSanity

> **检查区块头:**
1. 检查POW工作量证明
2. MedianTimeSource检查区块的时间是否超过最大偏移，中间时间也是区块库一个重要属性，产生分布式一致时钟。

```go
// checkBlockHeaderSanity performs some preliminary checks on a block header to
// ensure it is sane before continuing with processing.  These checks are
// context free.
//
// The flags do not modify the behavior of this function directly, however they
// are needed to pass along to checkProofOfWork.
func checkBlockHeaderSanity(header *wire.BlockHeader, powLimit *big.Int, timeSource MedianTimeSource, flags BehaviorFlags) error {
    // Ensure the proof of work bits in the block header is in min/max range
    // and the block hash is less than the target value described by the
    // bits.
    err := checkProofOfWork(header, powLimit, flags)
    if err != nil {
        return err
    }

    // A block timestamp must not have a greater precision than one second.
    // This check is necessary because Go time.Time values support
    // nanosecond precision whereas the consensus rules only apply to
    // seconds and it's much nicer to deal with standard Go time values
    // instead of converting to seconds everywhere.
    if !header.Timestamp.Equal(time.Unix(header.Timestamp.Unix(), 0)) {
        str := fmt.Sprintf("block timestamp of %v has a higher "+
            "precision than one second", header.Timestamp)
        return ruleError(ErrInvalidTime, str)
    }

    // Ensure the block time is not too far in the future.
    maxTimestamp := timeSource.AdjustedTime().Add(time.Second *
        MaxTimeOffsetSeconds)
    if header.Timestamp.After(maxTimestamp) {
        str := fmt.Sprintf("block timestamp of %v is too far in the "+
            "future", header.Timestamp)
        return ruleError(ErrTimeTooNew, str)
    }

    return nil
}
```

> checkProofOfWork

**header.Bits为区块生成难度值，是工作量证明重要参数。
系统是通过动态调整目标难度，控制区块的生成间隔时间。因此，这时主要是用hash生成一个big.Int值，与header.Bits比较。也就是说hash的值要小于header.Bits才是合法区块。Cmp是比较方法。**

```go
// checkProofOfWork ensures the block header bits which indicate the target
// difficulty is in min/max range and that the block hash is less than the
// target difficulty as claimed.
//
// The flags modify the behavior of this function as follows:
//  - BFNoPoWCheck: The check to ensure the block hash is less than the target
//    difficulty is not performed.
func checkProofOfWork(header *wire.BlockHeader, powLimit *big.Int, flags BehaviorFlags) error {
    // The target difficulty must be larger than zero.
    target := CompactToBig(header.Bits)
    if target.Sign() <= 0 {
        str := fmt.Sprintf("block target difficulty of %064x is too low",
            target)
        return ruleError(ErrUnexpectedDifficulty, str)
    }

    // The target difficulty must be less than the maximum allowed.
    if target.Cmp(powLimit) > 0 {
        str := fmt.Sprintf("block target difficulty of %064x is "+
            "higher than max of %064x", target, powLimit)
        return ruleError(ErrUnexpectedDifficulty, str)
    }

    // The block hash must be less than the claimed target unless the flag
    // to avoid proof of work checks is set.
    if flags&BFNoPoWCheck != BFNoPoWCheck {
        // The block hash must be less than the claimed target.
        hash := header.BlockHash()
        hashNum := HashToBig(&hash)
        if hashNum.Cmp(target) > 0 {
            str := fmt.Sprintf("block hash of %064x is higher than "+
                "expected max of %064x", hashNum, target)
            return ruleError(ErrHighHash, str)
        }
    }

    return nil
}
```

> MedianTimeSource

**MedianTimeSource provides a mechanism to add several time samples which are used to determine a median time which is then used as an offset to the local clock.**

```go
type MedianTimeSource interface {
    // AdjustedTime returns the current time adjusted by the median time
    // offset as calculated from the time samples added by AddTimeSample.
    AdjustedTime() time.Time

    // AddTimeSample adds a time sample that is used when determining the
    // median time of the added samples.
    AddTimeSample(id string, timeVal time.Time)

    // Offset returns the number of seconds to adjust the local clock based
    // upon the median of the time samples added by AddTimeData.
    Offset() time.Duration
}
```

#### 1.2.2.2. IsCoinBase

这个方法用于检查区块第一个交易是否为coinbase。这个交易是个特殊的交易，里面包括了miner挖矿的奖金和交易费用。

```go
// IsCoinBase determines whether or not a transaction is a coinbase.  A coinbase
// is a special transaction created by miners that has no inputs.  This is
// represented in the block chain by a transaction with a single input that has
// a previous output transaction index set to the maximum value along with a
// zero hash.
//
// This function only differs from IsCoinBaseTx in that it works with a higher
// level util transaction as opposed to a raw wire transaction.
func IsCoinBase(tx *btcutil.Tx) bool {
    return IsCoinBaseTx(tx.MsgTx())
}

func IsCoinBaseTx(msgTx *wire.MsgTx) bool {
    // A coin base must only have one transaction input.
    if len(msgTx.TxIn) != 1 {
        return false
    }

    // The previous output of a coin base must have a max value index and
    // a zero hash.
    prevOut := &msgTx.TxIn[0].PreviousOutPoint
    if prevOut.Index != math.MaxUint32 || prevOut.Hash != zeroHash {
        return false
    }

    return true
}

```

#### 1.2.2.3. CheckTransactionSanity

>CheckTransactionSanity说明检查区块中包含的交易是否合规。通过下面的代码可以看出这些检查只是些基本的逻辑范围之类的检查，不会也无法检查TxIn中的OutPoint是否已经被花费（在其它区块中）。

```go
// CheckTransactionSanity performs some preliminary checks on a transaction to
// ensure it is sane.  These checks are context free.
func CheckTransactionSanity(tx *btcutil.Tx) error {
    // A transaction must have at least one input.
    msgTx := tx.MsgTx()
    if len(msgTx.TxIn) == 0 {
        return ruleError(ErrNoTxInputs, "transaction has no inputs")
    }

    // A transaction must have at least one output.
    if len(msgTx.TxOut) == 0 {
        return ruleError(ErrNoTxOutputs, "transaction has no outputs")
    }

    // A transaction must not exceed the maximum allowed block payload when
    // serialized.
    serializedTxSize := tx.MsgTx().SerializeSizeStripped()
    if serializedTxSize > MaxBlockBaseSize {
        str := fmt.Sprintf("serialized transaction is too big - got "+
            "%d, max %d", serializedTxSize, MaxBlockBaseSize)
        return ruleError(ErrTxTooBig, str)
    }

    // Ensure the transaction amounts are in range.  Each transaction
    // output must not be negative or more than the max allowed per
    // transaction.  Also, the total of all outputs must abide by the same
    // restrictions.  All amounts in a transaction are in a unit value known
    // as a satoshi.  One bitcoin is a quantity of satoshi as defined by the
    // SatoshiPerBitcoin constant.
    var totalSatoshi int64
    for _, txOut := range msgTx.TxOut {
        satoshi := txOut.Value
        if satoshi < 0 {
            str := fmt.Sprintf("transaction output has negative "+
                "value of %v", satoshi)
            return ruleError(ErrBadTxOutValue, str)
        }
        if satoshi > btcutil.MaxSatoshi {
            str := fmt.Sprintf("transaction output value of %v is "+
                "higher than max allowed value of %v", satoshi,
                btcutil.MaxSatoshi)
            return ruleError(ErrBadTxOutValue, str)
        }

        // Two's complement int64 overflow guarantees that any overflow
        // is detected and reported.  This is impossible for Bitcoin, but
        // perhaps possible if an alt increases the total money supply.
        totalSatoshi += satoshi
        if totalSatoshi < 0 {
            str := fmt.Sprintf("total value of all transaction "+
                "outputs exceeds max allowed value of %v",
                btcutil.MaxSatoshi)
            return ruleError(ErrBadTxOutValue, str)
        }
        if totalSatoshi > btcutil.MaxSatoshi {
            str := fmt.Sprintf("total value of all transaction "+
                "outputs is %v which is higher than max "+
                "allowed value of %v", totalSatoshi,
                btcutil.MaxSatoshi)
            return ruleError(ErrBadTxOutValue, str)
        }
    }

    // Check for duplicate transaction inputs.
    existingTxOut := make(map[wire.OutPoint]struct{})
    for _, txIn := range msgTx.TxIn {
        if _, exists := existingTxOut[txIn.PreviousOutPoint]; exists {
            return ruleError(ErrDuplicateTxInputs, "transaction "+
                "contains duplicate inputs")
        }
        existingTxOut[txIn.PreviousOutPoint] = struct{}{}
    }

    // Coinbase script length must be between min and max length.
    if IsCoinBase(tx) {
        slen := len(msgTx.TxIn[0].SignatureScript)
        if slen < MinCoinbaseScriptLen || slen > MaxCoinbaseScriptLen {
            str := fmt.Sprintf("coinbase transaction script length "+
                "of %d is out of range (min: %d, max: %d)",
                slen, MinCoinbaseScriptLen, MaxCoinbaseScriptLen)
            return ruleError(ErrBadCoinbaseScriptLen, str)
        }
    } else {
        // Previous transaction outputs referenced by the inputs to this
        // transaction must not be null.
        for _, txIn := range msgTx.TxIn {
            if isNullOutpoint(&txIn.PreviousOutPoint) {
                return ruleError(ErrBadTxInput, "transaction "+
                    "input refers to previous output that "+
                    "is null")
            }
        }
    }

    return nil
}
```

### 1.2.3. findPreviousCheckpoint

这个方法用于从当前节点所有已经保存过的区块中查找到最近的一个在checkpoint点上的区块，checkpoint是在chaincfg中配置的，在前面章节有介绍过。

- nextCheckpoint:是当前节点下一个检查点，而且是没有得到的。
- checkpointNode:是当前节点已经包含的最近的检查点的区块。

如果以上两个属性都为空，就会去查找这两个值。在bestchain中的最近区块没有超过checkpoints中最大的检查点之前。nextCheckpoint是一定可以找到的，就算只有一个区块，而checkpointNode不一定有。如果达到了nextCheckpoint(b.bestChain.Tip().height == b.nextCheckpoint.Height)，就把当前nextCheckpoint的节点设置成checkpointNode，然后开始查找下一个nextCheckpoint，
当达到最后一个checkpoint(b.bestChain.Tip().height == b.checkpoints[numCheckpoints-1].Height)时，nextCheckpoint被设置成空。以后就永远都固定在最后一个checkpoint点的区块了。除非代码更新，添加新的checkpoint。

>逻辑如下：

```go
// findPreviousCheckpoint finds the most recent checkpoint that is already
// available in the downloaded portion of the block chain and returns the
// associated block node.  It returns nil if a checkpoint can't be found (this
// should really only happen for blocks before the first checkpoint).
//
// This function MUST be called with the chain lock held (for reads).
func (b *BlockChain) findPreviousCheckpoint() (*blockNode, error) {
    if !b.HasCheckpoints() {
        return nil, nil
    }

    // Perform the initial search to find and cache the latest known
    // checkpoint if the best chain is not known yet or we haven't already
    // previously searched.
    checkpoints := b.checkpoints
    numCheckpoints := len(checkpoints)
    if b.checkpointNode == nil && b.nextCheckpoint == nil {
        // Loop backwards through the available checkpoints to find one
        // that is already available.
        for i := numCheckpoints - 1; i >= 0; i-- {
            node := b.index.LookupNode(checkpoints[i].Hash)
            if node == nil || !b.bestChain.Contains(node) {
                continue
            }

            // Checkpoint found.  Cache it for future lookups and
            // set the next expected checkpoint accordingly.
            b.checkpointNode = node
            if i < numCheckpoints-1 {
                b.nextCheckpoint = &checkpoints[i+1]
            }
            return b.checkpointNode, nil
        }

        // No known latest checkpoint.  This will only happen on blocks
        // before the first known checkpoint.  So, set the next expected
        // checkpoint to the first checkpoint and return the fact there
        // is no latest known checkpoint block.
        b.nextCheckpoint = &checkpoints[0]
        return nil, nil
    }

    // At this point we've already searched for the latest known checkpoint,
    // so when there is no next checkpoint, the current checkpoint lockin
    // will always be the latest known checkpoint.
    if b.nextCheckpoint == nil {
        return b.checkpointNode, nil
    }

    // When there is a next checkpoint and the height of the current best
    // chain does not exceed it, the current checkpoint lockin is still
    // the latest known checkpoint.
    if b.bestChain.Tip().height < b.nextCheckpoint.Height {
        return b.checkpointNode, nil
    }

    // We've reached or exceeded the next checkpoint height.  Note that
    // once a checkpoint lockin has been reached, forks are prevented from
    // any blocks before the checkpoint, so we don't have to worry about the
    // checkpoint going away out from under us due to a chain reorganize.

    // Cache the latest known checkpoint for future lookups.  Note that if
    // this lookup fails something is very wrong since the chain has already
    // passed the checkpoint which was verified as accurate before inserting
    // it.
    checkpointNode := b.index.LookupNode(b.nextCheckpoint.Hash)
    if checkpointNode == nil {
        return nil, AssertError(fmt.Sprintf("findPreviousCheckpoint "+
            "failed lookup of known good block node %s",
            b.nextCheckpoint.Hash))
    }
    b.checkpointNode = checkpointNode

    // Set the next expected checkpoint.
    checkpointIndex := -1
    for i := numCheckpoints - 1; i >= 0; i-- {
        if checkpoints[i].Hash.IsEqual(b.nextCheckpoint.Hash) {
            checkpointIndex = i
            break
        }
    }
    b.nextCheckpoint = nil
    if checkpointIndex != -1 && checkpointIndex < numCheckpoints-1 {
        b.nextCheckpoint = &checkpoints[checkpointIndex+1]
    }

    return b.checkpointNode, nil
}
```

#### 1.2.3.1. checkpointNode比较

回到ProcessBlock中，如果查找的checkpointNode不为空，就可以这个检查点为标准，检查生成的区块的时间及难度数据，因为新的区块的难度值是一定比requiredTarget要小的，同时，新区块的Timestamp也是一定在checkpointNode之后的。

**注意：难度值越小，说明区块生成难度越大。**

> requiredTarget计算逻辑：

newTarget.Mul(newTarget, adjustmentFactor)，每次把newTarget*4。直到durationVal小于等于0.

```go
duration := blockHeader.Timestamp.Sub(checkpointTime)
requiredTarget := CompactToBig(b.calcEasiestDifficulty(
            checkpointNode.bits, duration))

// ---------CompactToBig是转换方法可以忽略

// calcEasiestDifficulty calculates the easiest possible difficulty that a block
// can have given starting difficulty bits and a duration.  It is mainly used to
// verify that claimed proof of work by a block is sane as compared to a
// known good checkpoint.
func (b *BlockChain) calcEasiestDifficulty(bits uint32, duration time.Duration) uint32 {
    // Convert types used in the calculations below.
    durationVal := int64(duration / time.Second)
    adjustmentFactor := big.NewInt(b.chainParams.RetargetAdjustmentFactor)

    // The test network rules allow minimum difficulty blocks after more
    // than twice the desired amount of time needed to generate a block has
    // elapsed.
    if b.chainParams.ReduceMinDifficulty {
        reductionTime := int64(b.chainParams.MinDiffReductionTime /
            time.Second)
        if durationVal > reductionTime {
            return b.chainParams.PowLimitBits
        }
    }

    // Since easier difficulty equates to higher numbers, the easiest
    // difficulty for a given duration is the largest value possible given
    // the number of retargets for the duration and starting difficulty
    // multiplied by the max adjustment factor.
    newTarget := CompactToBig(bits)
    for durationVal > 0 && newTarget.Cmp(b.chainParams.PowLimit) < 0 {
        newTarget.Mul(newTarget, adjustmentFactor)
        durationVal -= b.maxRetargetTimespan
    }

    // Limit new value to the proof of work limit.
    if newTarget.Cmp(b.chainParams.PowLimit) > 0 {
        newTarget.Set(b.chainParams.PowLimit)
    }

    return BigToCompact(newTarget)
}
```

> 参数值：

系统每过14天左右就要检查一下区块生成的难度，使生成一个区块的时间在TargetTimePerBlock左右。

```go
// TargetTimespan is the desired amount of time that should elapse
// before the block difficulty requirement is examined to determine how
// it should be changed in order to maintain the desired block
// generation rate.
TargetTimespan time.Duration


TargetTimespan:           time.Hour * 24 * 14, // 14 days
TargetTimePerBlock:       time.Minute * 10,    // 10 minutes
RetargetAdjustmentFactor: 4,                   // 25% less, 400% more
```

```go
// 创建BlockChain时的代码：

params := config.ChainParams
targetTimespan := int64(params.TargetTimespan / time.Second)
targetTimePerBlock := int64(params.TargetTimePerBlock / time.Second)
adjustmentFactor := params.RetargetAdjustmentFactor
b := BlockChain{
    ...
    minRetargetTimespan: targetTimespan / adjustmentFactor,
    maxRetargetTimespan: targetTimespan * adjustmentFactor,
    blocksPerRetarget:   int32(targetTimespan / targetTimePerBlock),
    ...
}
```

### 1.2.4. addOrphanBlock

在检查orphanblock之前，会先得到prevHashExists，也就是前一个区块是否存在（blockExists不重复说明）。**如果为false.就会添加当前block到orphans中，并且结束ProcessBlock处理。**

1. 会移走过期的孤儿节点，并且设置一个oldestOrphan（最旧的孤儿节点）
2. 如果orphans达到上限，就移走oldestOrphan
3. 添加到orphans
4. 维护进prevOrphans，prevOrphans用于查找当前区块的儿子节点。因为可能会有多个孤儿区块的前一个区块是相同的，所以这里b.prevOrphans[*prevHash]是一个数组。

```go
// addOrphanBlock adds the passed block (which is already determined to be
// an orphan prior calling this function) to the orphan pool.  It lazily cleans
// up any expired blocks so a separate cleanup poller doesn't need to be run.
// It also imposes a maximum limit on the number of outstanding orphan
// blocks and will remove the oldest received orphan block if the limit is
// exceeded.
func (b *BlockChain) addOrphanBlock(block *btcutil.Block) {
    // Remove expired orphan blocks.
    for _, oBlock := range b.orphans {
        if time.Now().After(oBlock.expiration) {
            b.removeOrphanBlock(oBlock)
            continue
        }

        // Update the oldest orphan block pointer so it can be discarded
        // in case the orphan pool fills up.
        if b.oldestOrphan == nil || oBlock.expiration.Before(b.oldestOrphan.expiration) {
            b.oldestOrphan = oBlock
        }
    }

    // Limit orphan blocks to prevent memory exhaustion.
    if len(b.orphans)+1 > maxOrphanBlocks {
        // Remove the oldest orphan to make room for the new one.
        b.removeOrphanBlock(b.oldestOrphan)
        b.oldestOrphan = nil
    }

    // Protect concurrent access.  This is intentionally done here instead
    // of near the top since removeOrphanBlock does its own locking and
    // the range iterator is not invalidated by removing map entries.
    b.orphanLock.Lock()
    defer b.orphanLock.Unlock()

    // Insert the block into the orphan map with an expiration time
    // 1 hour from now.
    expiration := time.Now().Add(time.Hour)
    oBlock := &orphanBlock{
        block:      block,
        expiration: expiration,
    }
    b.orphans[*block.Hash()] = oBlock

    // Add to previous hash lookup index for faster dependency lookups.
    prevHash := &block.MsgBlock().Header.PrevBlock
    b.prevOrphans[*prevHash] = append(b.prevOrphans[*prevHash], oBlock)
}
```

### 1.2.5. maybeAcceptBlock

maybeAcceptBlock也是一个重点方法，用于检查区块是否在bestchain。并且保存区块到数据库

```go
// maybeAcceptBlock potentially accepts a block into the block chain and, if
// accepted, returns whether or not it is on the main chain.  It performs
// several validation checks which depend on its position within the block chain
// before adding it.  The block is expected to have already gone through
// ProcessBlock before calling this function with it.
//
// The flags are also passed to checkBlockContext and connectBestChain.  See
// their documentation for how the flags modify their behavior.
//
// This function MUST be called with the chain state lock held (for writes).
func (b *BlockChain) maybeAcceptBlock(block *btcutil.Block, flags BehaviorFlags) (bool, error) {
    // The height of this block is one more than the referenced previous
    // block.
    prevHash := &block.MsgBlock().Header.PrevBlock
    prevNode := b.index.LookupNode(prevHash)
    if prevNode == nil {
        str := fmt.Sprintf("previous block %s is unknown", prevHash)
        return false, ruleError(ErrPreviousBlockUnknown, str)
    } else if b.index.NodeStatus(prevNode).KnownInvalid() {
        str := fmt.Sprintf("previous block %s is known to be invalid", prevHash)
        return false, ruleError(ErrInvalidAncestorBlock, str)
    }

    blockHeight := prevNode.height + 1
    block.SetHeight(blockHeight)

    // The block must pass all of the validation rules which depend on the
    // position of the block within the block chain.
    err := b.checkBlockContext(block, prevNode, flags)
    if err != nil {
        return false, err
    }

    // Insert the block into the database if it's not already there.  Even
    // though it is possible the block will ultimately fail to connect, it
    // has already passed all proof-of-work and validity tests which means
    // it would be prohibitively expensive for an attacker to fill up the
    // disk with a bunch of blocks that fail to connect.  This is necessary
    // since it allows block download to be decoupled from the much more
    // expensive connection logic.  It also has some other nice properties
    // such as making blocks that never become part of the main chain or
    // blocks that fail to connect available for further analysis.
    err = b.db.Update(func(dbTx database.Tx) error {
        return dbStoreBlock(dbTx, block)
    })
    if err != nil {
        return false, err
    }

    // Create a new block node for the block and add it to the node index. Even
    // if the block ultimately gets connected to the main chain, it starts out
    // on a side chain.
    blockHeader := &block.MsgBlock().Header
    newNode := newBlockNode(blockHeader, prevNode)
    newNode.status = statusDataStored

    b.index.AddNode(newNode)
    err = b.index.flushToDB()
    if err != nil {
        return false, err
    }

    // Connect the passed block to the chain while respecting proper chain
    // selection according to the chain with the most proof of work.  This
    // also handles validation of the transaction scripts.
    isMainChain, err := b.connectBestChain(newNode, block, flags)
    if err != nil {
        return false, err
    }

    // Notify the caller that the new block was accepted into the block
    // chain.  The caller would typically want to react by relaying the
    // inventory to other peers.
    b.chainLock.Unlock()
    b.sendNotification(NTBlockAccepted, block)
    b.chainLock.Lock()

    return isMainChain, nil
}
```

>主要流程：

1. checkBlockContext(block, prevNode, flags)
2. dbStoreBlock(dbTx, block)
    1. 保存区块到数据据中
3. newBlockNode(blockHeader, prevNode)
4. index.AddNode(newNode)
5. connectBestChain(newNode, block, flags)
6. sendNotification(NTBlockAccepted, block)
    1. 发送NTBlockAccepted通知,在上一章中提过，发送这个通知，会传播这个区块。

#### 1.2.5.1. checkBlockContext

跳过前面几个很好理解的检查，看下checkBlockContext做了什么事。可以看到如果为BFFastAdd模式，只会调用checkBlockHeaderContext。

1. checkBlockHeaderContext 检查区块头内容。
2. IsFinalizedTransaction检查是否有交易是未完成的。检查lockTime和Sequence。
3. 当高度超过BIP0034Height时，在coinbase中会添加当前区块的高度。检查这个高度是否正确。
4. 检查segwit是否激活,如果激活了就验证segwit。
5. 检查高度是否超过MaxBlockWeight。

```go
// checkBlockContext peforms several validation checks on the block which depend
// on its position within the block chain.
//
// The flags modify the behavior of this function as follows:
//  - BFFastAdd: The transaction are not checked to see if they are finalized
//    and the somewhat expensive BIP0034 validation is not performed.
//
// The flags are also passed to checkBlockHeaderContext.  See its documentation
// for how the flags modify its behavior.
//
// This function MUST be called with the chain state lock held (for writes).
func (b *BlockChain) checkBlockContext(block *btcutil.Block, prevNode *blockNode, flags BehaviorFlags) error {
    // Perform all block header related validation checks.
    header := &block.MsgBlock().Header
    err := b.checkBlockHeaderContext(header, prevNode, flags)
    if err != nil {
        return err
    }

    fastAdd := flags&BFFastAdd == BFFastAdd
    if !fastAdd {
        // Obtain the latest state of the deployed CSV soft-fork in
        // order to properly guard the new validation behavior based on
        // the current BIP 9 version bits state.
        csvState, err := b.deploymentState(prevNode, chaincfg.DeploymentCSV)
        if err != nil {
            return err
        }

        // Once the CSV soft-fork is fully active, we'll switch to
        // using the current median time past of the past block's
        // timestamps for all lock-time based checks.
        blockTime := header.Timestamp
        if csvState == ThresholdActive {
            blockTime = prevNode.CalcPastMedianTime()
        }

        // The height of this block is one more than the referenced
        // previous block.
        blockHeight := prevNode.height + 1

        // Ensure all transactions in the block are finalized.
        for _, tx := range block.Transactions() {
            if !IsFinalizedTransaction(tx, blockHeight,
                blockTime) {

                str := fmt.Sprintf("block contains unfinalized "+
                    "transaction %v", tx.Hash())
                return ruleError(ErrUnfinalizedTx, str)
            }
        }

        // Ensure coinbase starts with serialized block heights for
        // blocks whose version is the serializedHeightVersion or newer
        // once a majority of the network has upgraded.  This is part of
        // BIP0034.
        if ShouldHaveSerializedBlockHeight(header) &&
            blockHeight >= b.chainParams.BIP0034Height {

            coinbaseTx := block.Transactions()[0]
            err := checkSerializedHeight(coinbaseTx, blockHeight)
            if err != nil {
                return err
            }
        }

        // Query for the Version Bits state for the segwit soft-fork
        // deployment. If segwit is active, we'll switch over to
        // enforcing all the new rules.
        segwitState, err := b.deploymentState(prevNode,
            chaincfg.DeploymentSegwit)
        if err != nil {
            return err
        }

        // If segwit is active, then we'll need to fully validate the
        // new witness commitment for adherence to the rules.
        if segwitState == ThresholdActive {
            // Validate the witness commitment (if any) within the
            // block.  This involves asserting that if the coinbase
            // contains the special commitment output, then this
            // merkle root matches a computed merkle root of all
            // the wtxid's of the transactions within the block. In
            // addition, various other checks against the
            // coinbase's witness stack.
            if err := ValidateWitnessCommitment(block); err != nil {
                return err
            }

            // Once the witness commitment, witness nonce, and sig
            // op cost have been validated, we can finally assert
            // that the block's weight doesn't exceed the current
            // consensus parameter.
            blockWeight := GetBlockWeight(block)
            if blockWeight > MaxBlockWeight {
                str := fmt.Sprintf("block's weight metric is "+
                    "too high - got %v, max %v",
                    blockWeight, MaxBlockWeight)
                return ruleError(ErrBlockWeightTooHigh, str)
            }
        }
    }

    return nil
}
```

#### 1.2.5.2. checkBlockHeaderContext

在checkBlockContext中会验证区块头内容。这里的验证就用到了BFFastAdd。为true时，会跳过一些验证。

1. 根据上一个区块算出下一个区块的难度。如果不相等说明区块无效。
2. 判断block.header.Timestamp是否在上一区块的medianTime之后。
3. 如果区块高度正好在一个checkpoint上，就判断hash是否相等。
4. 判断高度是否比上一个checkpointNode大
5. 版本更新点区块高度检查

```go
// checkBlockHeaderContext performs several validation checks on the block header
// which depend on its position within the block chain.
//
// The flags modify the behavior of this function as follows:
//  - BFFastAdd: All checks except those involving comparing the header against
//    the checkpoints are not performed.
//
// This function MUST be called with the chain state lock held (for writes).
func (b *BlockChain) checkBlockHeaderContext(header *wire.BlockHeader, prevNode *blockNode, flags BehaviorFlags) error {
    fastAdd := flags&BFFastAdd == BFFastAdd
    if !fastAdd {
        // Ensure the difficulty specified in the block header matches
        // the calculated difficulty based on the previous block and
        // difficulty retarget rules.
        expectedDifficulty, err := b.calcNextRequiredDifficulty(prevNode,
            header.Timestamp)
        if err != nil {
            return err
        }
        blockDifficulty := header.Bits
        if blockDifficulty != expectedDifficulty {
            str := "block difficulty of %d is not the expected value of %d"
            str = fmt.Sprintf(str, blockDifficulty, expectedDifficulty)
            return ruleError(ErrUnexpectedDifficulty, str)
        }

        // Ensure the timestamp for the block header is after the
        // median time of the last several blocks (medianTimeBlocks).
        medianTime := prevNode.CalcPastMedianTime()
        if !header.Timestamp.After(medianTime) {
            str := "block timestamp of %v is not after expected %v"
            str = fmt.Sprintf(str, header.Timestamp, medianTime)
            return ruleError(ErrTimeTooOld, str)
        }
    }

    // The height of this block is one more than the referenced previous
    // block.
    blockHeight := prevNode.height + 1

    // Ensure chain matches up to predetermined checkpoints.
    blockHash := header.BlockHash()
    if !b.verifyCheckpoint(blockHeight, &blockHash) {
        str := fmt.Sprintf("block at height %d does not match "+
            "checkpoint hash", blockHeight)
        return ruleError(ErrBadCheckpoint, str)
    }

    // Find the previous checkpoint and prevent blocks which fork the main
    // chain before it.  This prevents storage of new, otherwise valid,
    // blocks which build off of old blocks that are likely at a much easier
    // difficulty and therefore could be used to waste cache and disk space.
    checkpointNode, err := b.findPreviousCheckpoint()
    if err != nil {
        return err
    }
    if checkpointNode != nil && blockHeight < checkpointNode.height {
        str := fmt.Sprintf("block at height %d forks the main chain "+
            "before the previous checkpoint at height %d",
            blockHeight, checkpointNode.height)
        return ruleError(ErrForkTooOld, str)
    }

    // Reject outdated block versions once a majority of the network
    // has upgraded.  These were originally voted on by BIP0034,
    // BIP0065, and BIP0066. 
    params := b.chainParams
    if header.Version < 2 && blockHeight >= params.BIP0034Height ||
        header.Version < 3 && blockHeight >= params.BIP0066Height ||
        header.Version < 4 && blockHeight >= params.BIP0065Height {

        str := "new blocks with version %d are no longer valid"
        str = fmt.Sprintf(str, header.Version)
        return ruleError(ErrBlockVersionTooOld, str)
    }

    return nil
}
```

### 1.2.6. dbStoreBlock之block.Bytes()

dbStoreBlock 会先检查它是否存在，然后调用dbTx存储区块。我们主要看下这个block是如何序列化为bytes。

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

// This function is part of the database.Tx interface implementation.
func (tx *transaction) StoreBlock(block *btcutil.Block) error {
    // Ensure transaction state is valid.
    if err := tx.checkClosed(); err != nil {
        return err
    }

    // Ensure the transaction is writable.
    if !tx.writable {
        str := "store block requires a writable database transaction"
        return makeDbErr(database.ErrTxNotWritable, str, nil)
    }

    // Reject the block if it already exists.
    blockHash := block.Hash()
    if tx.hasBlock(blockHash) {
        str := fmt.Sprintf("block %s already exists", blockHash)
        return makeDbErr(database.ErrBlockExists, str, nil)
    }

    blockBytes, err := block.Bytes()
    if err != nil {
        str := fmt.Sprintf("failed to get serialized bytes for block %s",
            blockHash)
        return makeDbErr(database.ErrDriverSpecific, str, err)
    }

    // Add the block to be stored to the list of pending blocks to store
    // when the transaction is committed.  Also, add it to pending blocks
    // map so it is easy to determine the block is pending based on the
    // block hash.
    if tx.pendingBlocks == nil {
        tx.pendingBlocks = make(map[chainhash.Hash]int)
    }
    tx.pendingBlocks[*blockHash] = len(tx.pendingBlockData)
    tx.pendingBlockData = append(tx.pendingBlockData, pendingBlock{
        hash:  blockHash,
        bytes: blockBytes,
    })
    log.Tracef("Added block %s to pending blocks", blockHash)

    return nil
}
```

ffldb自己实现的事务，所以数据不会直接保存，而是先放到缓存中。我们看下block.Bytes()逻辑。

```go
// Bytes returns the serialized bytes for the Block.  This is equivalent to
// calling Serialize on the underlying wire.MsgBlock, however it caches the
// result so subsequent calls are more efficient.
func (b *Block) Bytes() ([]byte, error) {
    // Return the cached serialized bytes if it has already been generated.
    if len(b.serializedBlock) != 0 {
        return b.serializedBlock, nil
    }

    // Serialize the MsgBlock.
    w := bytes.NewBuffer(make([]byte, 0, b.msgBlock.SerializeSize()))
    err := b.msgBlock.Serialize(w)
    if err != nil {
        return nil, err
    }
    serializedBlock := w.Bytes()

    // Cache the serialized bytes and return them.
    b.serializedBlock = serializedBlock
    return serializedBlock, nil
}

func (msg *MsgBlock) Serialize(w io.Writer) error {
    // At the current time, there is no difference between the wire encoding
    // at protocol version 0 and the stable long-term storage format.  As
    // a result, make use of BtcEncode.
    //
    // Passing WitnessEncoding as the encoding type here indicates that
    // each of the transactions should be serialized using the witness
    // serialization structure defined in BIP0141.
    return msg.BtcEncode(w, 0, WitnessEncoding)
}

```

可以看到，这里使用的是带有Witness的序列化。

#### 1.2.6.1. 序列化常识

> **序列化一个基本类型，如uint32代码：**

```go
func (littleEndian) PutUint32(b []byte, v uint32) {
    _ = b[3] // early bounds check to guarantee safety of writes below
    b[0] = byte(v)
    b[1] = byte(v >> 8)
    b[2] = byte(v >> 16)
    b[3] = byte(v >> 24)
}

func (bigEndian) PutUint32(b []byte, v uint32) {
    _ = b[3] // early bounds check to guarantee safety of writes below
    b[0] = byte(v >> 24)
    b[1] = byte(v >> 16)
    b[2] = byte(v >> 8)
    b[3] = byte(v)
}
```

小头是依次从低到高取8位放到b[0]到b[3]。大头相反。

>**序列化Bytes**

```go
func WriteVarBytes(w io.Writer, pver uint32, bytes []byte) error {
    slen := uint64(len(bytes))
    err := WriteVarInt(w, pver, slen)
    if err != nil {
        return err
    }

    _, err = w.Write(bytes)
    return err
}
```

bytes由于大小不固定，因此要先写bytes的长度（占8bytes），然后再写bytes。

#### 1.2.6.2. BtcEncode

encode一个block。依次是处理header，然后处理交易。
结构如下：

- Version  4bytes
- PrevBlockHash 4bytes
- MerkleRootHash 4bytes
- Timestamp.Unix() 4bytes
- Bits 4bytes
- Nonce 4bytes
- 交易数量 8bytes
- 所有交易数据

```go
// BtcEncode encodes the receiver to w using the bitcoin protocol encoding.
// This is part of the Message interface implementation.
// See Serialize for encoding blocks to be stored to disk, such as in a
// database, as opposed to encoding blocks for the wire.
func (msg *MsgBlock) BtcEncode(w io.Writer, pver uint32, enc MessageEncoding) error {
    err := writeBlockHeader(w, pver, &msg.Header)
    if err != nil {
        return err
    }

    err = WriteVarInt(w, pver, uint64(len(msg.Transactions)))
    if err != nil {
        return err
    }

    for _, tx := range msg.Transactions {
        err = tx.BtcEncode(w, pver, enc)
        if err != nil {
            return err
        }
    }

    return nil
}
```

> **writeBlockHeader**

```go
// writeBlockHeader writes a bitcoin block header to w.  See Serialize for
// encoding block headers to be stored to disk, such as in a database, as
// opposed to encoding for the wire.
func writeBlockHeader(w io.Writer, pver uint32, bh *BlockHeader) error {
    sec := uint32(bh.Timestamp.Unix())
    return writeElements(w, bh.Version, &bh.PrevBlock, &bh.MerkleRoot,
        sec, bh.Bits, bh.Nonce)
}

// writeElements writes multiple items to w.  It is equivalent to multiple
// calls to writeElement.
func writeElements(w io.Writer, elements ...interface{}) error {
    for _, element := range elements {
        err := writeElement(w, element)
        if err != nil {
            return err
        }
    }
    return nil
}
```

> **MsgTx.BtcEncode**

>**带witness数据的交encode如下：**

- tx.Version 4bytes
- witessMarkerBytes 2bytes
- TxIn数量 8bytes
- 所有的TxIn
    - txin.OutPointHash 4bytes
    - txin.Index 4bytes
    - txin.SignatureScript Nbytes
    - txin.Sequence 4bytes
- TxOut数量 8bytes
- 所有的TxOut
    - txout.Value 8bytes
    - txout.PkScript Nbytes

> **tips:在一个带有witness的txIn中，SignatureScript为空。**

```go
// witnessMarkerBytes are a pair of bytes specific to the witness encoding. If
// this sequence is encoutered, then it indicates a transaction has iwtness
// data. The first byte is an always 0x00 marker byte, which allows decoders to
// distinguish a serialized transaction with witnesses from a regular (legacy)
// one. The second byte is the Flag field, which at the moment is always 0x01,
// but may be extended in the future to accommodate auxiliary non-committed
// fields.
var witessMarkerBytes = []byte{0x00, 0x01}

// BtcEncode encodes the receiver to w using the bitcoin protocol encoding.
// This is part of the Message interface implementation.
// See Serialize for encoding transactions to be stored to disk, such as in a
// database, as opposed to encoding transactions for the wire.
func (msg *MsgTx) BtcEncode(w io.Writer, pver uint32, enc MessageEncoding) error {
    err := binarySerializer.PutUint32(w, littleEndian, uint32(msg.Version))
    if err != nil {
        return err
    }

    // If the encoding version is set to WitnessEncoding, and the Flags
    // field for the MsgTx aren't 0x00, then this indicates the transaction
    // is to be encoded using the new witness inclusionary structure
    // defined in BIP0144.
    doWitness := enc == WitnessEncoding && msg.HasWitness()
    if doWitness {
        // After the txn's Version field, we include two additional
        // bytes specific to the witness encoding. The first byte is an
        // always 0x00 marker byte, which allows decoders to
        // distinguish a serialized transaction with witnesses from a
        // regular (legacy) one. The second byte is the Flag field,
        // which at the moment is always 0x01, but may be extended in
        // the future to accommodate auxiliary non-committed fields.
        if _, err := w.Write(witessMarkerBytes); err != nil {
            return err
        }
    }

    count := uint64(len(msg.TxIn))
    err = WriteVarInt(w, pver, count)
    if err != nil {
        return err
    }

    for _, ti := range msg.TxIn {
        err = writeTxIn(w, pver, msg.Version, ti)
        if err != nil {
            return err
        }
    }

    count = uint64(len(msg.TxOut))
    err = WriteVarInt(w, pver, count)
    if err != nil {
        return err
    }

    for _, to := range msg.TxOut {
        err = WriteTxOut(w, pver, msg.Version, to)
        if err != nil {
            return err
        }
    }

    // If this transaction is a witness transaction, and the witness
    // encoded is desired, then encode the witness for each of the inputs
    // within the transaction.
    if doWitness {
        for _, ti := range msg.TxIn {
            err = writeTxWitness(w, pver, msg.Version, ti.Witness)
            if err != nil {
                return err
            }
        }
    }

    return binarySerializer.PutUint32(w, littleEndian, msg.LockTime)
}

// writeTxIn encodes ti to the bitcoin protocol encoding for a transaction
// input (TxIn) to w.
func writeTxIn(w io.Writer, pver uint32, version int32, ti *TxIn) error {
    err := writeOutPoint(w, pver, version, &ti.PreviousOutPoint)
    if err != nil {
        return err
    }

    err = WriteVarBytes(w, pver, ti.SignatureScript)
    if err != nil {
        return err
    }

    return binarySerializer.PutUint32(w, littleEndian, ti.Sequence)
}

// writeOutPoint encodes op to the bitcoin protocol encoding for an OutPoint
// to w.
func writeOutPoint(w io.Writer, pver uint32, version int32, op *OutPoint) error {
    _, err := w.Write(op.Hash[:])
    if err != nil {
        return err
    }

    return binarySerializer.PutUint32(w, littleEndian, op.Index)
}

func WriteTxOut(w io.Writer, pver uint32, version int32, to *TxOut) error {
    err := binarySerializer.PutUint64(w, littleEndian, uint64(to.Value))
    if err != nil {
        return err
    }

    return WriteVarBytes(w, pver, to.PkScript)
}

// writeTxWitness encodes the bitcoin protocol encoding for a transaction
// input's witness into to w.
func writeTxWitness(w io.Writer, pver uint32, version int32, wit [][]byte) error {
    err := WriteVarInt(w, pver, uint64(len(wit)))
    if err != nil {
        return err
    }
    for _, item := range wit {
        err = WriteVarBytes(w, pver, item)
        if err != nil {
            return err
        }
    }
    return nil
}
```

### 1.2.7. newBlockNode

newBlockNode会创建一个新的blockNode对象。这个对象会维护在内存中。这个对象不包括交易数据。

```go
// newBlockNode returns a new block node for the given block header and parent
// node, calculating the height and workSum from the respective fields on the
// parent. This function is NOT safe for concurrent access.
func newBlockNode(blockHeader *wire.BlockHeader, parent *blockNode) *blockNode {
    var node blockNode
    initBlockNode(&node, blockHeader, parent)
    return &node
}

// initBlockNode initializes a block node from the given header and parent node,
// calculating the height and workSum from the respective fields on the parent.
// This function is NOT safe for concurrent access.  It must only be called when
// initially creating a node.
func initBlockNode(node *blockNode, blockHeader *wire.BlockHeader, parent *blockNode) {
    *node = blockNode{
        hash:       blockHeader.BlockHash(),
        workSum:    CalcWork(blockHeader.Bits),
        version:    blockHeader.Version,
        bits:       blockHeader.Bits,
        nonce:      blockHeader.Nonce,
        timestamp:  blockHeader.Timestamp.Unix(),
        merkleRoot: blockHeader.MerkleRoot,
    }
    if parent != nil {
        node.parent = parent
        node.height = parent.height + 1
        node.workSum = node.workSum.Add(parent.workSum, node.workSum)
    }
}

// CalcWork calculates a work value from difficulty bits.  Bitcoin increases
// the difficulty for generating a block by decreasing the value which the
// generated hash must be less than.  This difficulty target is stored in each
// block header using a compact representation as described in the documentation
// for CompactToBig.  The main chain is selected by choosing the chain that has
// the most proof of work (highest difficulty).  Since a lower target difficulty
// value equates to higher actual difficulty, the work value which will be
// accumulated must be the inverse of the difficulty.  Also, in order to avoid
// potential division by zero and really small floating point numbers, the
// result adds 1 to the denominator and multiplies the numerator by 2^256.
func CalcWork(bits uint32) *big.Int {
    // Return a work value of zero if the passed difficulty bits represent
    // a negative number. Note this should not happen in practice with valid
    // blocks, but an invalid block could trigger it.
    difficultyNum := CompactToBig(bits)
    if difficultyNum.Sign() <= 0 {
        return big.NewInt(0)
    }

    // (1 << 256) / (difficultyNum + 1)
    denominator := new(big.Int).Add(difficultyNum, bigOne)
    return new(big.Int).Div(oneLsh256, denominator)
}
```

### 1.2.8. AddNode

上面创建了新的blocknode之后，调用AddNode添加到索引中，然后会调用flushToDB保存到db中。

```go
// AddNode adds the provided node to the block index and marks it as dirty.
// Duplicate entries are not checked so it is up to caller to avoid adding them.
//
// This function is safe for concurrent access.
func (bi *blockIndex) AddNode(node *blockNode) {
    bi.Lock()
    bi.addNode(node)
    bi.dirty[node] = struct{}{}
    bi.Unlock()
}

func (bi *blockIndex) addNode(node *blockNode) {
    bi.index[node.hash] = node
}

// flushToDB writes all dirty block nodes to the database. If all writes
// succeed, this clears the dirty set.
func (bi *blockIndex) flushToDB() error {
    bi.Lock()
    if len(bi.dirty) == 0 {
        bi.Unlock()
        return nil
    }

    err := bi.db.Update(func(dbTx database.Tx) error {
        for node := range bi.dirty {
            err := dbStoreBlockNode(dbTx, node)
            if err != nil {
                return err
            }
        }
        return nil
    })

    // If write was successful, clear the dirty set.
    if err == nil {
        bi.dirty = make(map[*blockNode]struct{})
    }

    bi.Unlock()
    return err
}

// dbStoreBlockNode stores the block header and validation status to the block
// index bucket. This overwrites the current entry if there exists one.
func dbStoreBlockNode(dbTx database.Tx, node *blockNode) error {
    // Serialize block data to be stored.
    w := bytes.NewBuffer(make([]byte, 0, blockHdrSize+1))
    header := node.Header()
    err := header.Serialize(w)
    if err != nil {
        return err
    }
    err = w.WriteByte(byte(node.status))
    if err != nil {
        return err
    }
    value := w.Bytes()

    // Write block header data to block index bucket.
    blockIndexBucket := dbTx.Metadata().Bucket(blockIndexBucketName)
    key := blockIndexKey(&node.hash, uint32(node.height))
    return blockIndexBucket.Put(key, value)
}
```

>通过上面的代码可以看出来，它就干了两件事

1. 维护到内存的索引bi.index[node.hash] = node
2. node保存到blockIndexBucket中


## 1.3. 通用方法说明

### 1.3.1. BuildMerkleTreeStore

>区块头中包括所有交易生成的MerkleTree root节点。同时MerkleTree也是blockchain技术中很重要的一个点。

MerkleTree是满二叉树。由于交易的hash全部在树的根节点中，可能会不足，所以要先计算出树的根节点数量。而二叉树的根的数量就是2的n-1次方，n为高度。nextPoT就是根节点数量。得到根节点数量就很容易算出arraySize。

>在生成树之前，要先计算交易的hash值，并添加到数组中merkles，其中有三个类型:

1. coinbase 为zeroHash
2. witness(隔离见证)交易，会生成带见证数据的hash
3. 正常hash

在根生成结束之后offset := nextPoT，说明右边空的节点直接当成空处理。比如只有三个节点

最后从数组倒数第二层开始生成根节点的父节点数据，也就是从offset开始。比如处理到h1和h2时，会生成h12。处理到h3和h4时，如果h4为空，就当h4值等于h3，生成h34。假如有6个交易，也就是说在生成h78时，由于h7为空，所以h78直接为空。

```go
// BuildMerkleTreeStore creates a merkle tree from a slice of transactions,
// stores it using a linear array, and returns a slice of the backing array.  A
// linear array was chosen as opposed to an actual tree structure since it uses
// about half as much memory.  The following describes a merkle tree and how it
// is stored in a linear array.
//
// A merkle tree is a tree in which every non-leaf node is the hash of its
// children nodes.  A diagram depicting how this works for bitcoin transactions
// where h(x) is a double sha256 follows:
//
//	         root = h1234 = h(h12 + h34)
//	        /                           \
//	  h12 = h(h1 + h2)            h34 = h(h3 + h4)
//	   /            \              /            \
//	h1 = h(tx1)  h2 = h(tx2)    h3 = h(tx3)  h4 = h(tx4)
//
// The above stored as a linear array is as follows:
//
// 	[h1 h2 h3 h4 h12 h34 root]
//
// As the above shows, the merkle root is always the last element in the array.
//
// The number of inputs is not always a power of two which results in a
// balanced tree structure as above.  In that case, parent nodes with no
// children are also zero and parent nodes with only a single left node
// are calculated by concatenating the left node with itself before hashing.
// Since this function uses nodes that are pointers to the hashes, empty nodes
// will be nil.
//
// The additional bool parameter indicates if we are generating the merkle tree
// using witness transaction id's rather than regular transaction id's. This
// also presents an additional case wherein the wtxid of the coinbase transaction
// is the zeroHash.
func BuildMerkleTreeStore(transactions []*btcutil.Tx, witness bool) []*chainhash.Hash {
    // Calculate how many entries are required to hold the binary merkle
    // tree as a linear array and create an array of that size.
    nextPoT := nextPowerOfTwo(len(transactions))
    arraySize := nextPoT*2 - 1
    merkles := make([]*chainhash.Hash, arraySize)

    // Create the base transaction hashes and populate the array with them.
    for i, tx := range transactions {
        // If we're computing a witness merkle root, instead of the
        // regular txid, we use the modified wtxid which includes a
        // transaction's witness data within the digest. Additionally,
        // the coinbase's wtxid is all zeroes.
        switch {
        case witness && i == 0:
            var zeroHash chainhash.Hash
            merkles[i] = &zeroHash
        case witness:
            wSha := tx.MsgTx().WitnessHash()
            merkles[i] = &wSha
        default:
            merkles[i] = tx.Hash()
        }

    }

    // Start the array offset after the last transaction and adjusted to the
    // next power of two.
    offset := nextPoT
    for i := 0; i < arraySize-1; i += 2 {
        switch {
        // When there is no left child node, the parent is nil too.
        case merkles[i] == nil:
            merkles[offset] = nil

        // When there is no right child, the parent is generated by
        // hashing the concatenation of the left child with itself.
        case merkles[i+1] == nil:
            newHash := HashMerkleBranches(merkles[i], merkles[i])
            merkles[offset] = newHash

        // The normal case sets the parent node to the double sha256
        // of the concatentation of the left and right children.
        default:
            newHash := HashMerkleBranches(merkles[i], merkles[i+1])
            merkles[offset] = newHash
        }
        offset++
    }

    return merkles
}
```

>看下BtcEncode中有没有Witness处理区别，前部分，都差别不大，只是在版本后面添加了一个witessMarkerBytes标记。真正的witness数据是在txin和txout都写完之后，最后添加进入的。

### 1.3.2. calcNextRequiredDifficulty

```go
// calcNextRequiredDifficulty calculates the required difficulty for the block
// after the passed previous block node based on the difficulty retarget rules.
// This function differs from the exported CalcNextRequiredDifficulty in that
// the exported version uses the current best chain as the previous block node
// while this function accepts any block node.
func (b *BlockChain) calcNextRequiredDifficulty(lastNode *blockNode, newBlockTime time.Time) (uint32, error) {
    // Genesis block.
    if lastNode == nil {
        return b.chainParams.PowLimitBits, nil
    }

    // Return the previous block's difficulty requirements if this block
    // is not at a difficulty retarget interval.
    if (lastNode.height+1)%b.blocksPerRetarget != 0 {
        // For networks that support it, allow special reduction of the
        // required difficulty once too much time has elapsed without
        // mining a block.
        if b.chainParams.ReduceMinDifficulty {
            // Return minimum difficulty when more than the desired
            // amount of time has elapsed without mining a block.
            reductionTime := int64(b.chainParams.MinDiffReductionTime /
                time.Second)
            allowMinTime := lastNode.timestamp + reductionTime
            if newBlockTime.Unix() > allowMinTime {
                return b.chainParams.PowLimitBits, nil
            }

            // The block was mined within the desired timeframe, so
            // return the difficulty for the last block which did
            // not have the special minimum difficulty rule applied.
            return b.findPrevTestNetDifficulty(lastNode), nil
        }

        // For the main network (or any unrecognized networks), simply
        // return the previous block's difficulty requirements.
        return lastNode.bits, nil
    }

    // Get the block node at the previous retarget (targetTimespan days
    // worth of blocks).
    firstNode := lastNode.RelativeAncestor(b.blocksPerRetarget - 1)
    if firstNode == nil {
        return 0, AssertError("unable to obtain previous retarget block")
    }

    // Limit the amount of adjustment that can occur to the previous
    // difficulty.
    actualTimespan := lastNode.timestamp - firstNode.timestamp
    adjustedTimespan := actualTimespan
    if actualTimespan < b.minRetargetTimespan {
        adjustedTimespan = b.minRetargetTimespan
    } else if actualTimespan > b.maxRetargetTimespan {
        adjustedTimespan = b.maxRetargetTimespan
    }

    // Calculate new target difficulty as:
    //  currentDifficulty * (adjustedTimespan / targetTimespan)
    // The result uses integer division which means it will be slightly
    // rounded down.  Bitcoind also uses integer division to calculate this
    // result.
    oldTarget := CompactToBig(lastNode.bits)
    newTarget := new(big.Int).Mul(oldTarget, big.NewInt(adjustedTimespan))
    targetTimeSpan := int64(b.chainParams.TargetTimespan / time.Second)
    newTarget.Div(newTarget, big.NewInt(targetTimeSpan))

    // Limit new value to the proof of work limit.
    if newTarget.Cmp(b.chainParams.PowLimit) > 0 {
        newTarget.Set(b.chainParams.PowLimit)
    }

    // Log new target difficulty and return it.  The new target logging is
    // intentionally converting the bits back to a number instead of using
    // newTarget since conversion to the compact representation loses
    // precision.
    newTargetBits := BigToCompact(newTarget)
    log.Debugf("Difficulty retarget at block height %d", lastNode.height+1)
    log.Debugf("Old target %08x (%064x)", lastNode.bits, oldTarget)
    log.Debugf("New target %08x (%064x)", newTargetBits, CompactToBig(newTargetBits))
    log.Debugf("Actual timespan %v, adjusted timespan %v, target timespan %v",
        time.Duration(actualTimespan)*time.Second,
        time.Duration(adjustedTimespan)*time.Second,
        b.chainParams.TargetTimespan)

    return newTargetBits, nil
}
```

### 1.3.3. IsFinalizedTransaction

如果lockTime小于LockTimeThreshold，表示锁定的是区块高度，小于这个高度说明交易还没有生效。同理，大于LockTimeThreshold表示时间。

```go
const LockTimeThreshold = 5e8

// IsFinalizedTransaction determines whether or not a transaction is finalized.
func IsFinalizedTransaction(tx *btcutil.Tx, blockHeight int32, blockTime time.Time) bool {
    msgTx := tx.MsgTx()

    // Lock time of zero means the transaction is finalized.
    lockTime := msgTx.LockTime
    if lockTime == 0 {
        return true
    }

    // The lock time field of a transaction is either a block height at
    // which the transaction is finalized or a timestamp depending on if the
    // value is before the txscript.LockTimeThreshold.  When it is under the
    // threshold it is a block height.
    blockTimeOrHeight := int64(0)
    if lockTime < txscript.LockTimeThreshold {
        blockTimeOrHeight = int64(blockHeight)
    } else {
        blockTimeOrHeight = blockTime.Unix()
    }
    if int64(lockTime) < blockTimeOrHeight {
        return true
    }

    // At this point, the transaction's lock time hasn't occurred yet, but
    // the transaction might still be finalized if the sequence number
    // for all transaction inputs is maxed out.
    for _, txIn := range msgTx.TxIn {
        if txIn.Sequence != math.MaxUint32 {
            return false
        }
    }
    return true
}
```

### 1.3.4. ValidateWitnessCommitment

激活witness之后，coinbase中会有witness数据。跳过基本检查之后。会从coinbaseTx中取出witnessCommitment，如果没有找到，那么所有交易中都不能有witness数据。取出32位的witnessNonce。再次调用BuildMerkleTreeStore生成tree,不过这次是要带上交易中的witness。最后合并witnessMerkleRoot和witnessNonce生成摘要与witnessCommitment对比。

coinbaseTx中两个重要的内容：

1. witnessCommitment 在其中一个TxOut中
2. Witness 在 Tx[0]中

```go
// ValidateWitnessCommitment validates the witness commitment (if any) found
// within the coinbase transaction of the passed block.
func ValidateWitnessCommitment(blk *btcutil.Block) error {
    // If the block doesn't have any transactions at all, then we won't be
    // able to extract a commitment from the non-existent coinbase
    // transaction. So we exit early here.
    if len(blk.Transactions()) == 0 {
        str := "cannot validate witness commitment of block without " +
            "transactions"
        return ruleError(ErrNoTransactions, str)
    }

    coinbaseTx := blk.Transactions()[0]
    if len(coinbaseTx.MsgTx().TxIn) == 0 {
        return ruleError(ErrNoTxInputs, "transaction has no inputs")
    }

    witnessCommitment, witnessFound := ExtractWitnessCommitment(coinbaseTx)

    // If we can't find a witness commitment in any of the coinbase's
    // outputs, then the block MUST NOT contain any transactions with
    // witness data.
    if !witnessFound {
        for _, tx := range blk.Transactions() {
            msgTx := tx.MsgTx()
            if msgTx.HasWitness() {
                str := fmt.Sprintf("block contains transaction with witness" +
                    " data, yet no witness commitment present")
                return ruleError(ErrUnexpectedWitness, str)
            }
        }
        return nil
    }

    // At this point the block contains a witness commitment, so the
    // coinbase transaction MUST have exactly one witness element within
    // its witness data and that element must be exactly
    // CoinbaseWitnessDataLen bytes.
    coinbaseWitness := coinbaseTx.MsgTx().TxIn[0].Witness
    if len(coinbaseWitness) != 1 {
        str := fmt.Sprintf("the coinbase transaction has %d items in "+
            "its witness stack when only one is allowed",
            len(coinbaseWitness))
        return ruleError(ErrInvalidWitnessCommitment, str)
    }
    witnessNonce := coinbaseWitness[0]
    if len(witnessNonce) != CoinbaseWitnessDataLen {
        str := fmt.Sprintf("the coinbase transaction witness nonce "+
            "has %d bytes when it must be %d bytes",
            len(witnessNonce), CoinbaseWitnessDataLen)
        return ruleError(ErrInvalidWitnessCommitment, str)
    }

    // Finally, with the preliminary checks out of the way, we can check if
    // the extracted witnessCommitment is equal to:
    // SHA256(witnessMerkleRoot || witnessNonce). Where witnessNonce is the
    // coinbase transaction's only witness item.
    witnessMerkleTree := BuildMerkleTreeStore(blk.Transactions(), true)
    witnessMerkleRoot := witnessMerkleTree[len(witnessMerkleTree)-1]

    var witnessPreimage [chainhash.HashSize * 2]byte
    copy(witnessPreimage[:], witnessMerkleRoot[:])
    copy(witnessPreimage[chainhash.HashSize:], witnessNonce)

    computedCommitment := chainhash.DoubleHashB(witnessPreimage[:])
    if !bytes.Equal(computedCommitment, witnessCommitment) {
        str := fmt.Sprintf("witness commitment does not match: "+
            "computed %v, coinbase includes %v", computedCommitment,
            witnessCommitment)
        return ruleError(ErrWitnessCommitmentMismatch, str)
    }

    return nil
}
```