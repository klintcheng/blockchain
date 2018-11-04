# 1. 处理区块

<!-- TOC -->

- [1. 处理区块](#1-%E5%A4%84%E7%90%86%E5%8C%BA%E5%9D%97)
    - [1.1. 介绍](#11-%E4%BB%8B%E7%BB%8D)
        - [1.1.1. ProcessBlock](#111-processblock)
        - [1.1.2. 参数详解](#112-%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3)
    - [1.2. 内部逻辑](#12-%E5%86%85%E9%83%A8%E9%80%BB%E8%BE%91)
        - [1.2.1. blockExists](#121-blockexists)
        - [1.2.2. checkBlockSanity](#122-checkblocksanity)
            - [1.2.2.1. checkBlockHeaderSanity](#1221-checkblockheadersanity)
            - [1.2.2.2. IsCoinBase](#1222-iscoinbase)
            - [1.2.2.3. CheckTransactionSanity](#1223-checktransactionsanity)
        - [1.2.3. findPreviousCheckpoint](#123-findpreviouscheckpoint)
            - [1.2.3.1. 与checkpointNode比较](#1231-%E4%B8%8Echeckpointnode%E6%AF%94%E8%BE%83)
        - [1.2.4. addOrphanBlock](#124-addorphanblock)
        - [1.2.5. maybeAcceptBlock](#125-maybeacceptblock)
            - [1.2.5.1. checkBlockContext](#1251-checkblockcontext)
            - [1.2.5.2. checkBlockHeaderContext](#1252-checkblockheadercontext)
            - [1.2.5.3. dbStoreBlock之block.Bytes()](#1253-dbstoreblock%E4%B9%8Bblockbytes)
                - [1.2.5.3.1. BtcEncode](#12531-btcencode)
            - [1.2.5.4. newBlockNode](#1254-newblocknode)
            - [1.2.5.5. AddNode](#1255-addnode)
            - [1.2.5.6. connectBestChain[*]](#1256-connectbestchain)
                - [1.2.5.6.1. 连到主链](#12561-%E8%BF%9E%E5%88%B0%E4%B8%BB%E9%93%BE)
                - [1.2.5.6.2. 侧链变主链](#12562-%E4%BE%A7%E9%93%BE%E5%8F%98%E4%B8%BB%E9%93%BE)
        - [1.2.6. processOrphans](#126-processorphans)
    - [1.3. 通用方法说明](#13-%E9%80%9A%E7%94%A8%E6%96%B9%E6%B3%95%E8%AF%B4%E6%98%8E)
        - [1.3.1. BuildMerkleTreeStore](#131-buildmerkletreestore)
        - [1.3.2. calcNextRequiredDifficulty](#132-calcnextrequireddifficulty)
        - [1.3.3. IsFinalizedTransaction](#133-isfinalizedtransaction)
        - [1.3.4. ValidateWitnessCommitment](#134-validatewitnesscommitment)
        - [1.3.5. checkConnectBlock](#135-checkconnectblock)
        - [1.3.6. connectBlock](#136-connectblock)
        - [1.3.7. 序列化常识](#137-%E5%BA%8F%E5%88%97%E5%8C%96%E5%B8%B8%E8%AF%86)
        - [1.3.8. CalcPastMedianTime](#138-calcpastmediantime)

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

下面会列出一些重要的逻辑，其中一些明显的验证判断之类的会忽略，直接看代码更清楚。

### 1.2.1. blockExists

首先要验证这个区块是否已经在主链或者侧链上。

在blockExists这个方法体内，先验证index, blockchain.index在内存中维护一份索引，因此查询速度快。然后先在验证区块是否在blockIdxBucket中，这个过程是通过调用leveldb接口查询的。

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

这个方法是个非常重要的方法，验证区块是否正常。

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

其中重点的需要说明验证列举一下，在下面添加详解：

1. checkBlockHeaderSanity 验证区块头是否正常
2. IsCoinBase 验证第一个交易是否为CoinBase交易
3. CheckTransactionSanity 验证所有交易是否正常

#### 1.2.2.1. checkBlockHeaderSanity

> **验证区块头:**
1. 验证POW工作量证明
2. MedianTimeSource验证区块的时间是否超过最大偏移，中间时间也是区块库一个重要属性，产生分布式一致时钟。

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

这个方法用于验证区块第一个交易是否为coinbase。这个交易是个特殊的交易，里面包括了miner挖矿的奖金和交易费用。

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

>CheckTransactionSanity说明验证区块中包含的交易是否合规。通过下面的代码可以看出这些验证只是些基本的逻辑范围之类的验证，不会也无法验证TxIn中的OutPoint是否已经被花费（在其它区块中）。

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

这个方法用于从当前节点所有已经保存过的区块中查找到最近的一个在checkpoint点上的区块，checkpoint是在chaincfg中配置的，在前面章节有介绍过。

- nextCheckpoint:是当前节点下一个验证点，而且是没有得到的。
- checkpointNode:是当前节点已经包含的最近的验证点的区块。

如果以上两个属性都为空，就会去查找这两个值。在bestchain中的最近区块没有超过checkpoints中最大的验证点之前。nextCheckpoint是一定可以找到的，就算只有一个区块，而checkpointNode不一定有。如果达到了nextCheckpoint(b.bestChain.Tip().height == b.nextCheckpoint.Height)，就把当前nextCheckpoint的节点设置成checkpointNode，然后开始查找下一个nextCheckpoint，
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

#### 1.2.3.1. 与checkpointNode比较

回到ProcessBlock中，如果查找的checkpointNode不为空，就可以以这个验证点为标准，验证生成的区块的时间及难度数据，因为calcEasiestDifficulty用于计算最容易情况下的难度值，因此在这个验证点之后的区块的难度值是一定比这个计算的出来的难度(requiredTarget)要小的，同时，新区块的Timestamp也是一定在checkpointNode之后的。

```go
duration := blockHeader.Timestamp.Sub(checkpointTime)
requiredTarget := CompactToBig(b.calcEasiestDifficulty(
            checkpointNode.bits, duration))
```

**注意：难度值越小，说明区块生成难度越大。**

> requiredTarget计算逻辑：

```go

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

> 上面代码中引用的参数

系统每过14天左右就要验证一下区块生成的难度，使生成一个区块的时间在TargetTimePerBlock左右。

```go
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

在验证orphanblock之前，会先得到prevHashExists，也就是前一个区块是否存在（blockExists不重复说明）。**如果为false.就会添加当前block到orphans中，并且结束ProcessBlock处理。**

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

1. 会移走过期的孤儿节点，并且设置一个oldestOrphan（最旧的孤儿节点）
2. 如果orphans达到上限，就移走oldestOrphan
3. oBlock添加到orphans
4. 维护进prevOrphans，prevOrphans用于查找当前区块的儿子节点。因为可能会有多个孤儿区块的前一个区块是相同的，所以这里b.prevOrphans[*prevHash]是一个数组。

**注意：孤儿池中的区块最多保存1小时**

### 1.2.5. maybeAcceptBlock

maybeAcceptBlock potentially accepts a block into the block chain and, if accepted, returns whether or not it is on the main chain.  It performs several validation checks which depend on its position within the block chain before adding it.  The block is expected to have already gone through ProcessBlock before calling this function with it.  
The flags are also passed to checkBlockContext and connectBestChain.  See their documentation for how the flags modify their behavior.

```go
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
3. newBlockNode(blockHeader, prevNode)
4. index.AddNode(newNode)
5. connectBestChain(newNode, block, flags)
6. sendNotification(NTBlockAccepted, block)  
   **发送NTBlockAccepted通知,在上一章中提过，发送这个通知，会传播这个区块。**

#### 1.2.5.1. checkBlockContext

跳过前面几个很好理解的验证，看下checkBlockContext做了什么事。可以看到如果为BFFastAdd模式，只会调用checkBlockHeaderContext。

checkBlockContext peforms several validation checks on the block which depend on its position within the block chain.

```go
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

1. checkBlockHeaderContext 验证区块头内容。
2. IsFinalizedTransaction验证是否有交易是未完成的。验证lockTime和Sequence。
3. 当高度超过BIP0034Height时，在coinbase中会添加当前区块的高度。验证这个高度是否正确。
4. 验证segwit是否激活,如果激活了就验证segwit。
5. 验证高度是否超过MaxBlockWeight。

#### 1.2.5.2. checkBlockHeaderContext

在checkBlockContext中会验证区块头内容。这里的验证就用到了BFFastAdd。为true时，会跳过一些验证。

1. 根据上一个区块算出下一个区块的难度。如果不相等说明区块无效。
2. 判断block.header.Timestamp是否在上一区块的medianTime之后。
3. 如果区块高度正好在一个checkpoint上，就判断hash是否相等。
4. 判断高度是否比上一个checkpointNode大
5. 版本更新点区块高度验证

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

#### 1.2.5.3. dbStoreBlock之block.Bytes()

dbStoreBlock 会先验证它是否存在，然后调用dbTx存储区块。我们主要看下这个block是如何序列化为bytes。

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

##### 1.2.5.3.1. BtcEncode

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

#### 1.2.5.4. newBlockNode

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

#### 1.2.5.5. AddNode

上面创建了新的blocknode之后，调用AddNode添加到索引中，然后会调用flushToDB保存到db中。

**注意：无论node是否在主链，都会被添加到这个索引中。同时，系统启动时，会加载所有的区块头生成blockNode，添加到这个索引中**

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

#### 1.2.5.6. connectBestChain[*]

connectBestChain处理连接通过的block到chain中，主链或者侧链。因此这个方法非常重要。

> 一个区块添加到链中，主要有三种情况：

1. 在主链后面生成，处理区块。
2. 在侧链上，但是工作量无法超过主链。
3. 在侧链上，而且已经超过主链，添加此区块之后这条侧链变成了主链，重新组织。

```go
// connectBestChain handles connecting the passed block to the chain while
// respecting proper chain selection according to the chain with the most
// proof of work.  In the typical case, the new block simply extends the main
// chain.  However, it may also be extending (or creating) a side chain (fork)
// which may or may not end up becoming the main chain depending on which fork
// cumulatively has the most proof of work.  It returns whether or not the block
// ended up on the main chain (either due to extending the main chain or causing
// a reorganization to become the main chain).
//
// The flags modify the behavior of this function as follows:
//  - BFFastAdd: Avoids several expensive transaction validation operations.
//    This is useful when using checkpoints.
//
// This function MUST be called with the chain state lock held (for writes).
func (b *BlockChain) connectBestChain(node *blockNode, block *btcutil.Block, flags BehaviorFlags) (bool, error) {
    fastAdd := flags&BFFastAdd == BFFastAdd

    flushIndexState := func() {
        // Intentionally ignore errors writing updated node status to DB. If
        // it fails to write, it's not the end of the world. If the block is
        // valid, we flush in connectBlock and if the block is invalid, the
        // worst that can happen is we revalidate the block after a restart.
        if writeErr := b.index.flushToDB(); writeErr != nil {
            log.Warnf("Error flushing block index changes to disk: %v",
                writeErr)
        }
    }

    // We are extending the main (best) chain with a new block.  This is the
    // most common case.
    parentHash := &block.MsgBlock().Header.PrevBlock
    if parentHash.IsEqual(&b.bestChain.Tip().hash) {
        // Skip checks if node has already been fully validated.
        fastAdd = fastAdd || b.index.NodeStatus(node).KnownValid()

        // Perform several checks to verify the block can be connected
        // to the main chain without violating any rules and without
        // actually connecting the block.
        view := NewUtxoViewpoint()
        view.SetBestHash(parentHash)
        stxos := make([]SpentTxOut, 0, countSpentOutputs(block))
        if !fastAdd {
            err := b.checkConnectBlock(node, block, view, &stxos)
            if err == nil {
                b.index.SetStatusFlags(node, statusValid)
            } else if _, ok := err.(RuleError); ok {
                b.index.SetStatusFlags(node, statusValidateFailed)
            } else {
                return false, err
            }

            flushIndexState()

            if err != nil {
                return false, err
            }
        }

        // In the fast add case the code to check the block connection
        // was skipped, so the utxo view needs to load the referenced
        // utxos, spend them, and add the new utxos being created by
        // this block.
        if fastAdd {
            err := view.fetchInputUtxos(b.db, block)
            if err != nil {
                return false, err
            }
            err = view.connectTransactions(block, &stxos)
            if err != nil {
                return false, err
            }
        }

        // Connect the block to the main chain.
        err := b.connectBlock(node, block, view, stxos)
        if err != nil {
            // If we got hit with a rule error, then we'll mark
            // that status of the block as invalid and flush the
            // index state to disk before returning with the error.
            if _, ok := err.(RuleError); ok {
                b.index.SetStatusFlags(
                    node, statusValidateFailed,
                )
            }

            flushIndexState()

            return false, err
        }

        // If this is fast add, or this block node isn't yet marked as
        // valid, then we'll update its status and flush the state to
        // disk again.
        if fastAdd || !b.index.NodeStatus(node).KnownValid() {
            b.index.SetStatusFlags(node, statusValid)
            flushIndexState()
        }

        return true, nil
    }
    if fastAdd {
        log.Warnf("fastAdd set in the side chain case? %v\n",
            block.Hash())
    }

    // We're extending (or creating) a side chain, but the cumulative
    // work for this new side chain is not enough to make it the new chain.
    if node.workSum.Cmp(b.bestChain.Tip().workSum) <= 0 {
        // Log information about how the block is forking the chain.
        fork := b.bestChain.FindFork(node)
        if fork.hash.IsEqual(parentHash) {
            log.Infof("FORK: Block %v forks the chain at height %d"+
                "/block %v, but does not cause a reorganize",
                node.hash, fork.height, fork.hash)
        } else {
            log.Infof("EXTEND FORK: Block %v extends a side chain "+
                "which forks the chain at height %d/block %v",
                node.hash, fork.height, fork.hash)
        }

        return false, nil
    }

    // We're extending (or creating) a side chain and the cumulative work
    // for this new side chain is more than the old best chain, so this side
    // chain needs to become the main chain.  In order to accomplish that,
    // find the common ancestor of both sides of the fork, disconnect the
    // blocks that form the (now) old fork from the main chain, and attach
    // the blocks that form the new chain to the main chain starting at the
    // common ancenstor (the point where the chain forked).
    detachNodes, attachNodes := b.getReorganizeNodes(node)

    // Reorganize the chain.
    log.Infof("REORGANIZE: Block %v is causing a reorganize.", node.hash)
    err := b.reorganizeChain(detachNodes, attachNodes)

    // Either getReorganizeNodes or reorganizeChain could have made unsaved
    // changes to the block index, so flush regardless of whether there was an
    // error. The index would only be dirty if the block failed to connect, so
    // we can ignore any errors writing.
    if writeErr := b.index.flushToDB(); writeErr != nil {
        log.Warnf("Error flushing block index changes to disk: %v", writeErr)
    }

    return err == nil, err
}
```

**注意：flushIndexState失败时不会处理，如果验证通过，在connectBlock时会再次提交**

##### 1.2.5.6.1. 连到主链

当parentHash.IsEqual(&b.bestChain.Tip().hash)时，表示这个区块在主链中。这里设fastAdd=false，可以看到它其实只做了如下几件事：

1. 生成UtxoViewpoint对象,最终会保存到utxoBucket中。
2. checkConnectBlock验证区块，这个验证是最终的验证，也是最复杂的逻辑。[详见通用方法]
3. connectBlock连接到主链[详见通用方法]
4. 更新blockNode状态

##### 1.2.5.6.2. 侧链变主链

如果侧链变成主链，则要先找到分叉点，然后原来在主链分叉点之后的所有区块（detachNodes）交易失效，同时对新的分叉点之后主链区块（attachNodes）进行上链处理。

```go
// getReorganizeNodes finds the fork point between the main chain and the passed
// node and returns a list of block nodes that would need to be detached from
// the main chain and a list of block nodes that would need to be attached to
// the fork point (which will be the end of the main chain after detaching the
// returned list of block nodes) in order to reorganize the chain such that the
// passed node is the new end of the main chain.  The lists will be empty if the
// passed node is not on a side chain.
//
// This function may modify node statuses in the block index without flushing.
//
// This function MUST be called with the chain state lock held (for reads).
func (b *BlockChain) getReorganizeNodes(node *blockNode) (*list.List, *list.List) {
    attachNodes := list.New()
    detachNodes := list.New()

    // Do not reorganize to a known invalid chain. Ancestors deeper than the
    // direct parent are checked below but this is a quick check before doing
    // more unnecessary work.
    if b.index.NodeStatus(node.parent).KnownInvalid() {
        b.index.SetStatusFlags(node, statusInvalidAncestor)
        return detachNodes, attachNodes
    }

    // Find the fork point (if any) adding each block to the list of nodes
    // to attach to the main tree.  Push them onto the list in reverse order
    // so they are attached in the appropriate order when iterating the list
    // later.
    forkNode := b.bestChain.FindFork(node)
    invalidChain := false
    for n := node; n != nil && n != forkNode; n = n.parent {
        if b.index.NodeStatus(n).KnownInvalid() {
            invalidChain = true
            break
        }
        attachNodes.PushFront(n)
    }

    // If any of the node's ancestors are invalid, unwind attachNodes, marking
    // each one as invalid for future reference.
    if invalidChain {
        var next *list.Element
        for e := attachNodes.Front(); e != nil; e = next {
            next = e.Next()
            n := attachNodes.Remove(e).(*blockNode)
            b.index.SetStatusFlags(n, statusInvalidAncestor)
        }
        return detachNodes, attachNodes
    }

    // Start from the end of the main chain and work backwards until the
    // common ancestor adding each block to the list of nodes to detach from
    // the main chain.
    for n := b.bestChain.Tip(); n != nil && n != forkNode; n = n.parent {
        detachNodes.PushBack(n)
    }

    return detachNodes, attachNodes
}
```

```go
// reorganizeChain reorganizes the block chain by disconnecting the nodes in the
// detachNodes list and connecting the nodes in the attach list.  It expects
// that the lists are already in the correct order and are in sync with the
// end of the current best chain.  Specifically, nodes that are being
// disconnected must be in reverse order (think of popping them off the end of
// the chain) and nodes the are being attached must be in forwards order
// (think pushing them onto the end of the chain).
//
// This function may modify node statuses in the block index without flushing.
//
// This function MUST be called with the chain state lock held (for writes).
func (b *BlockChain) reorganizeChain(detachNodes, attachNodes *list.List) error {
    // Nothing to do if no reorganize nodes were provided.
    if detachNodes.Len() == 0 && attachNodes.Len() == 0 {
        return nil
    }

    // Ensure the provided nodes match the current best chain.
    tip := b.bestChain.Tip()
    if detachNodes.Len() != 0 {
        firstDetachNode := detachNodes.Front().Value.(*blockNode)
        if firstDetachNode.hash != tip.hash {
            return AssertError(fmt.Sprintf("reorganize nodes to detach are "+
                "not for the current best chain -- first detach node %v, "+
                "current chain %v", &firstDetachNode.hash, &tip.hash))
        }
    }

    // Ensure the provided nodes are for the same fork point.
    if attachNodes.Len() != 0 && detachNodes.Len() != 0 {
        firstAttachNode := attachNodes.Front().Value.(*blockNode)
        lastDetachNode := detachNodes.Back().Value.(*blockNode)
        if firstAttachNode.parent.hash != lastDetachNode.parent.hash {
            return AssertError(fmt.Sprintf("reorganize nodes do not have the "+
                "same fork point -- first attach parent %v, last detach "+
                "parent %v", &firstAttachNode.parent.hash,
                &lastDetachNode.parent.hash))
        }
    }

    // Track the old and new best chains heads.
    oldBest := tip
    newBest := tip

    // All of the blocks to detach and related spend journal entries needed
    // to unspend transaction outputs in the blocks being disconnected must
    // be loaded from the database during the reorg check phase below and
    // then they are needed again when doing the actual database updates.
    // Rather than doing two loads, cache the loaded data into these slices.
    detachBlocks := make([]*btcutil.Block, 0, detachNodes.Len())
    detachSpentTxOuts := make([][]SpentTxOut, 0, detachNodes.Len())
    attachBlocks := make([]*btcutil.Block, 0, attachNodes.Len())

    // Disconnect all of the blocks back to the point of the fork.  This
    // entails loading the blocks and their associated spent txos from the
    // database and using that information to unspend all of the spent txos
    // and remove the utxos created by the blocks.
    view := NewUtxoViewpoint()
    view.SetBestHash(&oldBest.hash)
    for e := detachNodes.Front(); e != nil; e = e.Next() {
        n := e.Value.(*blockNode)
        var block *btcutil.Block
        err := b.db.View(func(dbTx database.Tx) error {
            var err error
            block, err = dbFetchBlockByNode(dbTx, n)
            return err
        })
        if err != nil {
            return err
        }
        if n.hash != *block.Hash() {
            return AssertError(fmt.Sprintf("detach block node hash %v (height "+
                "%v) does not match previous parent block hash %v", &n.hash,
                n.height, block.Hash()))
        }

        // Load all of the utxos referenced by the block that aren't
        // already in the view.
        err = view.fetchInputUtxos(b.db, block)
        if err != nil {
            return err
        }

        // Load all of the spent txos for the block from the spend
        // journal.
        var stxos []SpentTxOut
        err = b.db.View(func(dbTx database.Tx) error {
            stxos, err = dbFetchSpendJournalEntry(dbTx, block)
            return err
        })
        if err != nil {
            return err
        }

        // Store the loaded block and spend journal entry for later.
        detachBlocks = append(detachBlocks, block)
        detachSpentTxOuts = append(detachSpentTxOuts, stxos)

        err = view.disconnectTransactions(b.db, block, stxos)
        if err != nil {
            return err
        }

        newBest = n.parent
    }

    // Set the fork point only if there are nodes to attach since otherwise
    // blocks are only being disconnected and thus there is no fork point.
    var forkNode *blockNode
    if attachNodes.Len() > 0 {
        forkNode = newBest
    }

    // Perform several checks to verify each block that needs to be attached
    // to the main chain can be connected without violating any rules and
    // without actually connecting the block.
    //
    // NOTE: These checks could be done directly when connecting a block,
    // however the downside to that approach is that if any of these checks
    // fail after disconnecting some blocks or attaching others, all of the
    // operations have to be rolled back to get the chain back into the
    // state it was before the rule violation (or other failure).  There are
    // at least a couple of ways accomplish that rollback, but both involve
    // tweaking the chain and/or database.  This approach catches these
    // issues before ever modifying the chain.
    for e := attachNodes.Front(); e != nil; e = e.Next() {
        n := e.Value.(*blockNode)

        var block *btcutil.Block
        err := b.db.View(func(dbTx database.Tx) error {
            var err error
            block, err = dbFetchBlockByNode(dbTx, n)
            return err
        })
        if err != nil {
            return err
        }

        // Store the loaded block for later.
        attachBlocks = append(attachBlocks, block)

        // Skip checks if node has already been fully validated. Although
        // checkConnectBlock gets skipped, we still need to update the UTXO
        // view.
        if b.index.NodeStatus(n).KnownValid() {
            err = view.fetchInputUtxos(b.db, block)
            if err != nil {
                return err
            }
            err = view.connectTransactions(block, nil)
            if err != nil {
                return err
            }

            newBest = n
            continue
        }

        // Notice the spent txout details are not requested here and
        // thus will not be generated.  This is done because the state
        // is not being immediately written to the database, so it is
        // not needed.
        //
        // In the case the block is determined to be invalid due to a
        // rule violation, mark it as invalid and mark all of its
        // descendants as having an invalid ancestor.
        err = b.checkConnectBlock(n, block, view, nil)
        if err != nil {
            if _, ok := err.(RuleError); ok {
                b.index.SetStatusFlags(n, statusValidateFailed)
                for de := e.Next(); de != nil; de = de.Next() {
                    dn := de.Value.(*blockNode)
                    b.index.SetStatusFlags(dn, statusInvalidAncestor)
                }
            }
            return err
        }
        b.index.SetStatusFlags(n, statusValid)

        newBest = n
    }

    // Reset the view for the actual connection code below.  This is
    // required because the view was previously modified when checking if
    // the reorg would be successful and the connection code requires the
    // view to be valid from the viewpoint of each block being connected or
    // disconnected.
    view = NewUtxoViewpoint()
    view.SetBestHash(&b.bestChain.Tip().hash)

    // Disconnect blocks from the main chain.
    for i, e := 0, detachNodes.Front(); e != nil; i, e = i+1, e.Next() {
        n := e.Value.(*blockNode)
        block := detachBlocks[i]

        // Load all of the utxos referenced by the block that aren't
        // already in the view.
        err := view.fetchInputUtxos(b.db, block)
        if err != nil {
            return err
        }

        // Update the view to unspend all of the spent txos and remove
        // the utxos created by the block.
        err = view.disconnectTransactions(b.db, block,
            detachSpentTxOuts[i])
        if err != nil {
            return err
        }

        // Update the database and chain state.
        err = b.disconnectBlock(n, block, view)
        if err != nil {
            return err
        }
    }

    // Connect the new best chain blocks.
    for i, e := 0, attachNodes.Front(); e != nil; i, e = i+1, e.Next() {
        n := e.Value.(*blockNode)
        block := attachBlocks[i]

        // Load all of the utxos referenced by the block that aren't
        // already in the view.
        err := view.fetchInputUtxos(b.db, block)
        if err != nil {
            return err
        }

        // Update the view to mark all utxos referenced by the block
        // as spent and add all transactions being created by this block
        // to it.  Also, provide an stxo slice so the spent txout
        // details are generated.
        stxos := make([]SpentTxOut, 0, countSpentOutputs(block))
        err = view.connectTransactions(block, &stxos)
        if err != nil {
            return err
        }

        // Update the database and chain state.
        err = b.connectBlock(n, block, view, stxos)
        if err != nil {
            return err
        }
    }

    // Log the point where the chain forked and old and new best chain
    // heads.
    if forkNode != nil {
        log.Infof("REORGANIZE: Chain forks at %v (height %v)", forkNode.hash,
            forkNode.height)
    }
    log.Infof("REORGANIZE: Old best chain head was %v (height %v)",
        &oldBest.hash, oldBest.height)
    log.Infof("REORGANIZE: New best chain head is %v (height %v)",
        newBest.hash, newBest.height)

    return nil
}
```

### 1.2.6. processOrphans

当一个区块连接到链上之后，就要验证是否有孤儿节点是它的子代。如果存在，就会调用maybeAcceptBlock处理。而且在processBlock中添加孤儿节点之前，已经通过了checkBlockSanity验证，这里不需要再处理。同时，新上链的孤儿区块又要添加到processHashes中等待处理。

```go
// processOrphans determines if there are any orphans which depend on the passed
// block hash (they are no longer orphans if true) and potentially accepts them.
// It repeats the process for the newly accepted blocks (to detect further
// orphans which may no longer be orphans) until there are no more.
//
// The flags do not modify the behavior of this function directly, however they
// are needed to pass along to maybeAcceptBlock.
//
// This function MUST be called with the chain state lock held (for writes).
func (b *BlockChain) processOrphans(hash *chainhash.Hash, flags BehaviorFlags) error {
    // Start with processing at least the passed hash.  Leave a little room
    // for additional orphan blocks that need to be processed without
    // needing to grow the array in the common case.
    processHashes := make([]*chainhash.Hash, 0, 10)
    processHashes = append(processHashes, hash)
    for len(processHashes) > 0 {
        // Pop the first hash to process from the slice.
        processHash := processHashes[0]
        processHashes[0] = nil // Prevent GC leak.
        processHashes = processHashes[1:]

        // Look up all orphans that are parented by the block we just
        // accepted.  This will typically only be one, but it could
        // be multiple if multiple blocks are mined and broadcast
        // around the same time.  The one with the most proof of work
        // will eventually win out.  An indexing for loop is
        // intentionally used over a range here as range does not
        // reevaluate the slice on each iteration nor does it adjust the
        // index for the modified slice.
        for i := 0; i < len(b.prevOrphans[*processHash]); i++ {
            orphan := b.prevOrphans[*processHash][i]
            if orphan == nil {
                log.Warnf("Found a nil entry at index %d in the "+
                    "orphan dependency list for block %v", i,
                    processHash)
                continue
            }

            // Remove the orphan from the orphan pool.
            orphanHash := orphan.block.Hash()
            b.removeOrphanBlock(orphan)
            i--

            // Potentially accept the block into the block chain.
            _, err := b.maybeAcceptBlock(orphan.block, flags)
            if err != nil {
                return err
            }

            // Add this block to the list of blocks to process so
            // any orphan blocks that depend on this block are
            // handled too.
            processHashes = append(processHashes, orphanHash)
        }
    }
    return nil
}
```


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

1. 如果当前区块高不在调整点
    1. ReduceMinDifficulty=false,直接返回上一个区块的bits难度。
2. 在调整点  
    oldTarget*adjustedTimespan/(TargetTimespan / time.Second)

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

激活witness之后，coinbase中会有witness数据。跳过基本验证之后。会从coinbaseTx中取出witnessCommitment，如果没有找到，那么所有交易中都不能有witness数据。取出32位的witnessNonce。再次调用BuildMerkleTreeStore生成tree,不过这次是要带上交易中的witness。最后合并witnessMerkleRoot和witnessNonce生成摘要与witnessCommitment对比。

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

### 1.3.5. checkConnectBlock

checkConnectBlock performs several checks to confirm connecting the passed block to the chain represented by the passed view does not violate any rules. In addition, the passed view is updated to spend all of the referenced outputs and add all of the new utxos created by block.  Thus, the view will represent the state of the chain as if the block were actually connected and consequently the best hash for the view is also updated to passed block. 

An example of some of the checks performed are ensuring connecting the block would not cause any duplicate transaction hashes for old transactions that aren't already fully spent, double spends, exceeding the maximum allowed signature operations per block, invalid values in relation to the expected block subsidy, or fail transaction script validation.

The CheckConnectBlockTemplate function makes use of this function to perform the bulk of its work.  The only difference is this function accepts a node which may or may not require reorganization to connect it to the main chain whereas CheckConnectBlockTemplate creates a new node which specifically connects to the end of the current main chain and then calls this function with that node.

```go
// This function MUST be called with the chain state lock held (for writes).
func (b *BlockChain) checkConnectBlock(node *blockNode, block *btcutil.Block, view *UtxoViewpoint, stxos *[]SpentTxOut) error {
    // If the side chain blocks end up in the database, a call to
    // CheckBlockSanity should be done here in case a previous version
    // allowed a block that is no longer valid.  However, since the
    // implementation only currently uses memory for the side chain blocks,
    // it isn't currently necessary.

    // The coinbase for the Genesis block is not spendable, so just return
    // an error now.
    if node.hash.IsEqual(b.chainParams.GenesisHash) {
        str := "the coinbase for the genesis block is not spendable"
        return ruleError(ErrMissingTxOut, str)
    }

    // Ensure the view is for the node being checked.
    parentHash := &block.MsgBlock().Header.PrevBlock
    if !view.BestHash().IsEqual(parentHash) {
        return AssertError(fmt.Sprintf("inconsistent view when "+
            "checking block connection: best hash is %v instead "+
            "of expected %v", view.BestHash(), parentHash))
    }

    // BIP0030 added a rule to prevent blocks which contain duplicate
    // transactions that 'overwrite' older transactions which are not fully
    // spent.  See the documentation for checkBIP0030 for more details.
    //
    // There are two blocks in the chain which violate this rule, so the
    // check must be skipped for those blocks.  The isBIP0030Node function
    // is used to determine if this block is one of the two blocks that must
    // be skipped.
    //
    // In addition, as of BIP0034, duplicate coinbases are no longer
    // possible due to its requirement for including the block height in the
    // coinbase and thus it is no longer possible to create transactions
    // that 'overwrite' older ones.  Therefore, only enforce the rule if
    // BIP0034 is not yet active.  This is a useful optimization because the
    // BIP0030 check is expensive since it involves a ton of cache misses in
    // the utxoset.
    if !isBIP0030Node(node) && (node.height < b.chainParams.BIP0034Height) {
        err := b.checkBIP0030(node, block, view)
        if err != nil {
            return err
        }
    }

    // Load all of the utxos referenced by the inputs for all transactions
    // in the block don't already exist in the utxo view from the database.
    //
    // These utxo entries are needed for verification of things such as
    // transaction inputs, counting pay-to-script-hashes, and scripts.
    err := view.fetchInputUtxos(b.db, block)
    if err != nil {
        return err
    }

    // BIP0016 describes a pay-to-script-hash type that is considered a
    // "standard" type.  The rules for this BIP only apply to transactions
    // after the timestamp defined by txscript.Bip16Activation.  See
    // https://en.bitcoin.it/wiki/BIP_0016 for more details.
    enforceBIP0016 := node.timestamp >= txscript.Bip16Activation.Unix()

    // Query for the Version Bits state for the segwit soft-fork
    // deployment. If segwit is active, we'll switch over to enforcing all
    // the new rules.
    segwitState, err := b.deploymentState(node.parent, chaincfg.DeploymentSegwit)
    if err != nil {
        return err
    }
    enforceSegWit := segwitState == ThresholdActive

    // The number of signature operations must be less than the maximum
    // allowed per block.  Note that the preliminary sanity checks on a
    // block also include a check similar to this one, but this check
    // expands the count to include a precise count of pay-to-script-hash
    // signature operations in each of the input transaction public key
    // scripts.
    transactions := block.Transactions()
    totalSigOpCost := 0
    for i, tx := range transactions {
        // Since the first (and only the first) transaction has
        // already been verified to be a coinbase transaction,
        // use i == 0 as an optimization for the flag to
        // countP2SHSigOps for whether or not the transaction is
        // a coinbase transaction rather than having to do a
        // full coinbase check again.
        sigOpCost, err := GetSigOpCost(tx, i == 0, view, enforceBIP0016,
            enforceSegWit)
        if err != nil {
            return err
        }

        // Check for overflow or going over the limits.  We have to do
        // this on every loop iteration to avoid overflow.
        lastSigOpCost := totalSigOpCost
        totalSigOpCost += sigOpCost
        if totalSigOpCost < lastSigOpCost || totalSigOpCost > MaxBlockSigOpsCost {
            str := fmt.Sprintf("block contains too many "+
                "signature operations - got %v, max %v",
                totalSigOpCost, MaxBlockSigOpsCost)
            return ruleError(ErrTooManySigOps, str)
        }
    }

    // Perform several checks on the inputs for each transaction.  Also
    // accumulate the total fees.  This could technically be combined with
    // the loop above instead of running another loop over the transactions,
    // but by separating it we can avoid running the more expensive (though
    // still relatively cheap as compared to running the scripts) checks
    // against all the inputs when the signature operations are out of
    // bounds.
    var totalFees int64
    for _, tx := range transactions {
        txFee, err := CheckTransactionInputs(tx, node.height, view,
            b.chainParams)
        if err != nil {
            return err
        }

        // Sum the total fees and ensure we don't overflow the
        // accumulator.
        lastTotalFees := totalFees
        totalFees += txFee
        if totalFees < lastTotalFees {
            return ruleError(ErrBadFees, "total fees for block "+
                "overflows accumulator")
        }

        // Add all of the outputs for this transaction which are not
        // provably unspendable as available utxos.  Also, the passed
        // spent txos slice is updated to contain an entry for each
        // spent txout in the order each transaction spends them.
        err = view.connectTransaction(tx, node.height, stxos)
        if err != nil {
            return err
        }
    }

    // The total output values of the coinbase transaction must not exceed
    // the expected subsidy value plus total transaction fees gained from
    // mining the block.  It is safe to ignore overflow and out of range
    // errors here because those error conditions would have already been
    // caught by checkTransactionSanity.
    var totalSatoshiOut int64
    for _, txOut := range transactions[0].MsgTx().TxOut {
        totalSatoshiOut += txOut.Value
    }
    expectedSatoshiOut := CalcBlockSubsidy(node.height, b.chainParams) +
        totalFees
    if totalSatoshiOut > expectedSatoshiOut {
        str := fmt.Sprintf("coinbase transaction for block pays %v "+
            "which is more than expected value of %v",
            totalSatoshiOut, expectedSatoshiOut)
        return ruleError(ErrBadCoinbaseValue, str)
    }

    // Don't run scripts if this node is before the latest known good
    // checkpoint since the validity is verified via the checkpoints (all
    // transactions are included in the merkle root hash and any changes
    // will therefore be detected by the next checkpoint).  This is a huge
    // optimization because running the scripts is the most time consuming
    // portion of block handling.
    checkpoint := b.LatestCheckpoint()
    runScripts := true
    if checkpoint != nil && node.height <= checkpoint.Height {
        runScripts = false
    }

    // Blocks created after the BIP0016 activation time need to have the
    // pay-to-script-hash checks enabled.
    var scriptFlags txscript.ScriptFlags
    if enforceBIP0016 {
        scriptFlags |= txscript.ScriptBip16
    }

    // Enforce DER signatures for block versions 3+ once the historical
    // activation threshold has been reached.  This is part of BIP0066.
    blockHeader := &block.MsgBlock().Header
    if blockHeader.Version >= 3 && node.height >= b.chainParams.BIP0066Height {
        scriptFlags |= txscript.ScriptVerifyDERSignatures
    }

    // Enforce CHECKLOCKTIMEVERIFY for block versions 4+ once the historical
    // activation threshold has been reached.  This is part of BIP0065.
    if blockHeader.Version >= 4 && node.height >= b.chainParams.BIP0065Height {
        scriptFlags |= txscript.ScriptVerifyCheckLockTimeVerify
    }

    // Enforce CHECKSEQUENCEVERIFY during all block validation checks once
    // the soft-fork deployment is fully active.
    csvState, err := b.deploymentState(node.parent, chaincfg.DeploymentCSV)
    if err != nil {
        return err
    }
    if csvState == ThresholdActive {
        // If the CSV soft-fork is now active, then modify the
        // scriptFlags to ensure that the CSV op code is properly
        // validated during the script checks bleow.
        scriptFlags |= txscript.ScriptVerifyCheckSequenceVerify

        // We obtain the MTP of the *previous* block in order to
        // determine if transactions in the current block are final.
        medianTime := node.parent.CalcPastMedianTime()

        // Additionally, if the CSV soft-fork package is now active,
        // then we also enforce the relative sequence number based
        // lock-times within the inputs of all transactions in this
        // candidate block.
        for _, tx := range block.Transactions() {
            // A transaction can only be included within a block
            // once the sequence locks of *all* its inputs are
            // active.
            sequenceLock, err := b.calcSequenceLock(node, tx, view,
                false)
            if err != nil {
                return err
            }
            if !SequenceLockActive(sequenceLock, node.height,
                medianTime) {
                str := fmt.Sprintf("block contains " +
                    "transaction whose input sequence " +
                    "locks are not met")
                return ruleError(ErrUnfinalizedTx, str)
            }
        }
    }

    // Enforce the segwit soft-fork package once the soft-fork has shifted
    // into the "active" version bits state.
    if enforceSegWit {
        scriptFlags |= txscript.ScriptVerifyWitness
        scriptFlags |= txscript.ScriptStrictMultiSig
    }

    // Now that the inexpensive checks are done and have passed, verify the
    // transactions are actually allowed to spend the coins by running the
    // expensive ECDSA signature check scripts.  Doing this last helps
    // prevent CPU exhaustion attacks.
    if runScripts {
        err := checkBlockScripts(block, view, scriptFlags, b.sigCache,
            b.hashCache)
        if err != nil {
            return err
        }
    }

    // Update the best hash for view to include this block since all of its
    // transactions have been connected.
    view.SetBestHash(&node.hash)

    return nil
}
```

1. checkBIP0030,软分叉规则验证。 [BIP0030 Detail](https://github.com/bitcoin/bips/blob/master/bip-0030.mediawiki)
2. fetchInputUtxos，读取出区块中的所有的utxo，只要有一个读取失败，就返回err
3. 验证所有交易签名操作之和是否超过限制
4. 验证每个交易，包括PreviousOutPoint指向的utxo是否存在且没有被花费，coinbase中的高度与区块高对比，花费Amount是否在正常,交易费是否正常等
5. 当所有的验证都通过之后，最好验证耗cpu的脚本。

### 1.3.6. connectBlock

```go
// connectBlock handles connecting the passed node/block to the end of the main
// (best) chain.
//
// This passed utxo view must have all referenced txos the block spends marked
// as spent and all of the new txos the block creates added to it.  In addition,
// the passed stxos slice must be populated with all of the information for the
// spent txos.  This approach is used because the connection validation that
// must happen prior to calling this function requires the same details, so
// it would be inefficient to repeat it.
//
// This function MUST be called with the chain state lock held (for writes).
func (b *BlockChain) connectBlock(node *blockNode, block *btcutil.Block,
    view *UtxoViewpoint, stxos []SpentTxOut) error {

    // Make sure it's extending the end of the best chain.
    prevHash := &block.MsgBlock().Header.PrevBlock
    if !prevHash.IsEqual(&b.bestChain.Tip().hash) {
        return AssertError("connectBlock must be called with a block " +
            "that extends the main chain")
    }

    // Sanity check the correct number of stxos are provided.
    if len(stxos) != countSpentOutputs(block) {
        return AssertError("connectBlock called with inconsistent " +
            "spent transaction out information")
    }

    // No warnings about unknown rules or versions until the chain is
    // current.
    if b.isCurrent() {
        // Warn if any unknown new rules are either about to activate or
        // have already been activated.
        if err := b.warnUnknownRuleActivations(node); err != nil {
            return err
        }

        // Warn if a high enough percentage of the last blocks have
        // unexpected versions.
        if err := b.warnUnknownVersions(node); err != nil {
            return err
        }
    }

    // Write any block status changes to DB before updating best state.
    err := b.index.flushToDB()
    if err != nil {
        return err
    }

    // Generate a new best state snapshot that will be used to update the
    // database and later memory if all database updates are successful.
    b.stateLock.RLock()
    curTotalTxns := b.stateSnapshot.TotalTxns
    b.stateLock.RUnlock()
    numTxns := uint64(len(block.MsgBlock().Transactions))
    blockSize := uint64(block.MsgBlock().SerializeSize())
    blockWeight := uint64(GetBlockWeight(block))
    state := newBestState(node, blockSize, blockWeight, numTxns,
        curTotalTxns+numTxns, node.CalcPastMedianTime())

    // Atomically insert info into the database.
    err = b.db.Update(func(dbTx database.Tx) error {
        // Update best block state.
        err := dbPutBestState(dbTx, state, node.workSum)
        if err != nil {
            return err
        }

        // Add the block hash and height to the block index which tracks
        // the main chain.
        err = dbPutBlockIndex(dbTx, block.Hash(), node.height)
        if err != nil {
            return err
        }

        // Update the utxo set using the state of the utxo view.  This
        // entails removing all of the utxos spent and adding the new
        // ones created by the block.
        err = dbPutUtxoView(dbTx, view)
        if err != nil {
            return err
        }

        // Update the transaction spend journal by adding a record for
        // the block that contains all txos spent by it.
        err = dbPutSpendJournalEntry(dbTx, block.Hash(), stxos)
        if err != nil {
            return err
        }

        // Allow the index manager to call each of the currently active
        // optional indexes with the block being connected so they can
        // update themselves accordingly.
        if b.indexManager != nil {
            err := b.indexManager.ConnectBlock(dbTx, block, stxos)
            if err != nil {
                return err
            }
        }

        return nil
    })
    if err != nil {
        return err
    }

    // Prune fully spent entries and mark all entries in the view unmodified
    // now that the modifications have been committed to the database.
    view.commit()

    // This node is now the end of the best chain.
    b.bestChain.SetTip(node)

    // Update the state for the best block.  Notice how this replaces the
    // entire struct instead of updating the existing one.  This effectively
    // allows the old version to act as a snapshot which callers can use
    // freely without needing to hold a lock for the duration.  See the
    // comments on the state variable for more details.
    b.stateLock.Lock()
    b.stateSnapshot = state
    b.stateLock.Unlock()

    // Notify the caller that the block was connected to the main chain.
    // The caller would typically want to react with actions such as
    // updating wallets.
    b.chainLock.Unlock()
    b.sendNotification(NTBlockConnected, block)
    b.chainLock.Lock()

    return nil
}
```

### 1.3.7. 序列化常识

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

### 1.3.8. CalcPastMedianTime

CalcPastMedianTime calculates the median time of the previous few blocks

- medianTimeBlocks: 11

```go
// prior to, and including, the block node.
//
// This function is safe for concurrent access.
func (node *blockNode) CalcPastMedianTime() time.Time {
    // Create a slice of the previous few block timestamps used to calculate
    // the median per the number defined by the constant medianTimeBlocks.
    timestamps := make([]int64, medianTimeBlocks)
    numNodes := 0
    iterNode := node
    for i := 0; i < medianTimeBlocks && iterNode != nil; i++ {
        timestamps[i] = iterNode.timestamp
        numNodes++

        iterNode = iterNode.parent
    }

    // Prune the slice to the actual number of available timestamps which
    // will be fewer than desired near the beginning of the block chain
    // and sort them.
    timestamps = timestamps[:numNodes]
    sort.Sort(timeSorter(timestamps))

    // NOTE: The consensus rules incorrectly calculate the median for even
    // numbers of blocks.  A true median averages the middle two elements
    // for a set with an even number of elements in it.   Since the constant
    // for the previous number of blocks to be used is odd, this is only an
    // issue for a few blocks near the beginning of the chain.  I suspect
    // this is an optimization even though the result is slightly wrong for
    // a few of the first blocks since after the first few blocks, there
    // will always be an odd number of blocks in the set per the constant.
    //
    // This code follows suit to ensure the same rules are used, however, be
    // aware that should the medianTimeBlocks constant ever be changed to an
    // even number, this code will be wrong.
    medianTimestamp := timestamps[numNodes/2]
    return time.Unix(medianTimestamp, 0)
}
```