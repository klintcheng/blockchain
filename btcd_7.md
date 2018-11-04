# 1. 同步管理

<!-- TOC -->

- [1. 同步管理](#1-%E5%90%8C%E6%AD%A5%E7%AE%A1%E7%90%86)
    - [1.1. 简介](#11-%E7%AE%80%E4%BB%8B)
        - [1.1.1. SyncManager](#111-syncmanager)
        - [1.1.2. 创建syncManager](#112-%E5%88%9B%E5%BB%BAsyncmanager)
    - [1.2. 启动过程](#12-%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B)
        - [1.2.1. blockHandler](#121-blockhandler)
    - [1.3. 同步区块](#13-%E5%90%8C%E6%AD%A5%E5%8C%BA%E5%9D%97)
        - [1.3.1. 触发同步](#131-%E8%A7%A6%E5%8F%91%E5%90%8C%E6%AD%A5)
        - [1.3.2. 开始同步](#132-%E5%BC%80%E5%A7%8B%E5%90%8C%E6%AD%A5)
            - [1.3.2.1. 请求区块头](#1321-%E8%AF%B7%E6%B1%82%E5%8C%BA%E5%9D%97%E5%A4%B4)
            - [1.3.2.2. 接收区块头](#1322-%E6%8E%A5%E6%94%B6%E5%8C%BA%E5%9D%97%E5%A4%B4)
            - [1.3.2.3. 请求区块](#1323-%E8%AF%B7%E6%B1%82%E5%8C%BA%E5%9D%97)
            - [1.3.2.4. 接收区块](#1324-%E6%8E%A5%E6%94%B6%E5%8C%BA%E5%9D%97)
            - [1.3.2.5. 处理收到的区块](#1325-%E5%A4%84%E7%90%86%E6%94%B6%E5%88%B0%E7%9A%84%E5%8C%BA%E5%9D%97)
        - [1.3.3. MsgGetHeaders处理](#133-msggetheaders%E5%A4%84%E7%90%86)
            - [1.3.3.1. LocateHeaders](#1331-locateheaders)
        - [1.3.4. MsgGetData请求处理](#134-msggetdata%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86)
            - [1.3.4.1. FetchBlock](#1341-fetchblock)
    - [1.4. 数据传播](#14-%E6%95%B0%E6%8D%AE%E4%BC%A0%E6%92%AD)
        - [1.4.1. 订阅通知](#141-%E8%AE%A2%E9%98%85%E9%80%9A%E7%9F%A5)
        - [1.4.2. 通知处理](#142-%E9%80%9A%E7%9F%A5%E5%A4%84%E7%90%86)
            - [1.4.2.1. peerNotifier实现](#1421-peernotifier%E5%AE%9E%E7%8E%B0)
        - [1.4.3. 库存广播](#143-%E5%BA%93%E5%AD%98%E5%B9%BF%E6%92%AD)
            - [1.4.3.1. handleRelayInvMsg](#1431-handlerelayinvmsg)
            - [1.4.3.2. OnInv](#1432-oninv)
            - [1.4.3.3. handleInvMsg](#1433-handleinvmsg)

<!-- /TOC -->

## 1.1. 简介

>SyncManager负责数据同步工作，包括启动时区块同步到最大高度。随时处理其它节点传播过来的交易或者区块，并且转发给其它节点。

### 1.1.1. SyncManager

```go
// SyncManager is used to communicate block related messages with peers. The
// SyncManager is started as by executing Start() in a goroutine. Once started,
// it selects peers to sync from and starts the initial block download. Once the
// chain is in sync, the SyncManager handles incoming block and header
// notifications and relays announcements of new blocks to peers.
type SyncManager struct {
    peerNotifier   PeerNotifier
    started        int32
    shutdown       int32
    chain          *blockchain.BlockChain
    txMemPool      *mempool.TxPool
    chainParams    *chaincfg.Params
    progressLogger *blockProgressLogger
    msgChan        chan interface{}
    wg             sync.WaitGroup
    quit           chan struct{}

    // These fields should only be accessed from the blockHandler thread
    rejectedTxns    map[chainhash.Hash]struct{}
    requestedTxns   map[chainhash.Hash]struct{}
    requestedBlocks map[chainhash.Hash]struct{}
    syncPeer        *peerpkg.Peer
    peerStates      map[*peerpkg.Peer]*peerSyncState

    // The following fields are used for headers-first mode.
    headersFirstMode bool
    headerList       *list.List
    startHeader      *list.Element
    nextCheckpoint   *chaincfg.Checkpoint

    // An optional fee estimator.
    feeEstimator *mempool.FeeEstimator
}
```

### 1.1.2. 创建syncManager

```go
// s=server
s.syncManager, err = netsync.New(&netsync.Config{
    PeerNotifier:       &s,
    Chain:              s.chain,
    TxMemPool:          s.txMemPool,
    ChainParams:        s.chainParams,
    DisableCheckpoints: cfg.DisableCheckpoints,
    MaxPeers:           cfg.MaxPeers,
    FeeEstimator:       s.feeEstimator,
})

// New constructs a new SyncManager. Use Start to begin processing asynchronous
// block, tx, and inv updates.
func New(config *Config) (*SyncManager, error) {
    sm := SyncManager{
        peerNotifier:    config.PeerNotifier,
        chain:           config.Chain,
        txMemPool:       config.TxMemPool,
        chainParams:     config.ChainParams,
        rejectedTxns:    make(map[chainhash.Hash]struct{}),
        requestedTxns:   make(map[chainhash.Hash]struct{}),
        requestedBlocks: make(map[chainhash.Hash]struct{}),
        peerStates:      make(map[*peerpkg.Peer]*peerSyncState),
        progressLogger:  newBlockProgressLogger("Processed", log),
        msgChan:         make(chan interface{}, config.MaxPeers*3),
        headerList:      list.New(),
        quit:            make(chan struct{}),
        feeEstimator:    config.FeeEstimator,
    }

    best := sm.chain.BestSnapshot()
    if !config.DisableCheckpoints {
        // Initialize the next checkpoint based on the current height.
        sm.nextCheckpoint = sm.findNextHeaderCheckpoint(best.Height)
        if sm.nextCheckpoint != nil {
            sm.resetHeaderState(&best.Hash, best.Height)
        }
    } else {
        log.Info("Checkpoints are disabled")
    }

    sm.chain.Subscribe(sm.handleBlockchainNotification)

    return &sm, nil
}

```

>这里会初始化区块头节点信息：

```go
// resetHeaderState sets the headers-first mode state to values appropriate for
// syncing from a new peer.
func (sm *SyncManager) resetHeaderState(newestHash *chainhash.Hash, newestHeight int32) {
    sm.headersFirstMode = false
    sm.headerList.Init()
    sm.startHeader = nil

    // When there is a next checkpoint, add an entry for the latest known
    // block into the header pool.  This allows the next downloaded header
    // to prove it links to the chain properly.
    if sm.nextCheckpoint != nil {
        node := headerNode{height: newestHeight, hash: newestHash}
        sm.headerList.PushBack(&node)
    }
}
```

## 1.2. 启动过程

Sync启动时会调用blockHandler()监听事件同步区块数。我们来看看它是如何同步的。

```go
// Start begins the core block handler which processes block and inv messages.
func (sm *SyncManager) Start() {
    // Already started?
    if atomic.AddInt32(&sm.started, 1) != 1 {
        return
    }

    log.Trace("Starting sync manager")
    sm.wg.Add(1)
    go sm.blockHandler()
}
```

### 1.2.1. blockHandler

blockHandler()为独立的goroutine，会一直等待msgChan消息，然后处理。

```go
// blockHandler is the main handler for the sync manager.  It must be run as a
// goroutine.  It processes block and inv messages in a separate goroutine
// from the peer handlers so the block (MsgBlock) messages are handled by a
// single thread without needing to lock memory data structures.  This is
// important because the sync manager controls which blocks are needed and how
// the fetching should proceed.
func (sm *SyncManager) blockHandler() {
out:
    for {
        select {
        case m := <-sm.msgChan:
            switch msg := m.(type) {
            case *newPeerMsg:
                sm.handleNewPeerMsg(msg.peer)

            case *txMsg:
                sm.handleTxMsg(msg)
                msg.reply <- struct{}{}

            case *blockMsg:
                sm.handleBlockMsg(msg)
                msg.reply <- struct{}{}

            case *invMsg:
                sm.handleInvMsg(msg)

            case *headersMsg:
                sm.handleHeadersMsg(msg)

            case *donePeerMsg:
                sm.handleDonePeerMsg(msg.peer)

            case getSyncPeerMsg:
                var peerID int32
                if sm.syncPeer != nil {
                    peerID = sm.syncPeer.ID()
                }
                msg.reply <- peerID

            case processBlockMsg:
                _, isOrphan, err := sm.chain.ProcessBlock(
                    msg.block, msg.flags)
                if err != nil {
                    msg.reply <- processBlockResponse{
                        isOrphan: false,
                        err:      err,
                    }
                }

                msg.reply <- processBlockResponse{
                    isOrphan: isOrphan,
                    err:      nil,
                }

            case isCurrentMsg:
                msg.reply <- sm.current()

            case pauseMsg:
                // Wait until the sender unpauses the manager.
                <-msg.unpause

            default:
                log.Warnf("Invalid message type in block "+
                    "handler: %T", msg)
            }

        case <-sm.quit:
            break out
        }
    }

    sm.wg.Done()
    log.Trace("Block handler done")
}
```

## 1.3. 同步区块

当一个新的节点添加之后，系统就会开始同步区块数据，直到最新的区块同步完成。在上一章节，我们在server.OnVersion中看到它会向Sync中添加一个节点。

### 1.3.1. 触发同步

```go
func (sp *serverPeer) OnVersion(_ *peer.Peer, msg *wire.MsgVersion) *wire.MsgReject {
    ...
    sp.server.syncManager.NewPeer(sp.Peer)
    ...
}
```

我们从这个方法入口开始，看下同步管理器处理流程。

```go
// NewPeer informs the sync manager of a newly active peer.
func (sm *SyncManager) NewPeer(peer *peerpkg.Peer) {
    // Ignore if we are shutting down.
    if atomic.LoadInt32(&sm.shutdown) != 0 {
        return
    }
    sm.msgChan <- &newPeerMsg{peer: peer}
}
```

创建一个newPeerMsg，并把它的指针写到msgChan，在上面我们已经看过了。所有的msgChan处理都是在一个goroutine中即blockHandler。当它收到这个消息时，会调用handleNewPeerMsg处理。

>handleNewPeerMsg

```go
// handleNewPeerMsg deals with new peers that have signalled they may
// be considered as a sync peer (they have already successfully negotiated).  It
// also starts syncing if needed.  It is invoked from the syncHandler goroutine.
func (sm *SyncManager) handleNewPeerMsg(peer *peerpkg.Peer) {
    // Ignore if in the process of shutting down.
    if atomic.LoadInt32(&sm.shutdown) != 0 {
        return
    }

    log.Infof("New valid peer %s (%s)", peer, peer.UserAgent())

    // Initialize the peer state
    isSyncCandidate := sm.isSyncCandidate(peer)
    sm.peerStates[peer] = &peerSyncState{
        syncCandidate:   isSyncCandidate,
        requestedTxns:   make(map[chainhash.Hash]struct{}),
        requestedBlocks: make(map[chainhash.Hash]struct{}),
    }

    // Start syncing by choosing the best candidate if needed.
    if isSyncCandidate && sm.syncPeer == nil {
        sm.startSync()
    }
}
```

>这个方法会做些同步前处理及判断：

1. 判断这个节点是否可以用于区块的同步
2. 初始化此节点的状态保存到peerStates
3. 如果条件满足，开始同步

> 大致看下判断是如何处理的

```go
// isSyncCandidate returns whether or not the peer is a candidate to consider
// syncing from.
func (sm *SyncManager) isSyncCandidate(peer *peerpkg.Peer) bool {
    // Typically a peer is not a candidate for sync if it's not a full node,
    // however regression test is special in that the regression tool is
    // not a full node and still needs to be considered a sync candidate.
    if sm.chainParams == &chaincfg.RegressionNetParams {
        // The peer is not a candidate if it's not coming from localhost
        // or the hostname can't be determined for some reason.
        host, _, err := net.SplitHostPort(peer.Addr())
        if err != nil {
            return false
        }

        if host != "127.0.0.1" && host != "localhost" {
            return false
        }
    } else {
        // The peer is not a candidate for sync if it's not a full
        // node. Additionally, if the segwit soft-fork package has
        // activated, then the peer must also be upgraded.
        segwitActive, err := sm.chain.IsDeploymentActive(chaincfg.DeploymentSegwit)
        if err != nil {
            log.Errorf("Unable to query for segwit "+
                "soft-fork state: %v", err)
        }
        nodeServices := peer.Services()
        if nodeServices&wire.SFNodeNetwork != wire.SFNodeNetwork ||
            (segwitActive && !peer.IsWitnessEnabled()) {
            return false
        }
    }

    // Candidate if all checks passed.
    return true
}
```

### 1.3.2. 开始同步

同步区块时，会先拿到头信息，然后再去取区块数据。

```go
// startSync will choose the best peer among the available candidate peers to
// download/sync the blockchain from.  When syncing is already running, it
// simply returns.  It also examines the candidates for any which are no longer
// candidates and removes them as needed.
func (sm *SyncManager) startSync() {
    // Return now if we're already syncing.
    if sm.syncPeer != nil {
        return
    }

    // Once the segwit soft-fork package has activated, we only
    // want to sync from peers which are witness enabled to ensure
    // that we fully validate all blockchain data.
    segwitActive, err := sm.chain.IsDeploymentActive(chaincfg.DeploymentSegwit)
    if err != nil {
        log.Errorf("Unable to query for segwit soft-fork state: %v", err)
        return
    }

    best := sm.chain.BestSnapshot()
    var bestPeer *peerpkg.Peer
    for peer, state := range sm.peerStates {
        if !state.syncCandidate {
            continue
        }

        if segwitActive && !peer.IsWitnessEnabled() {
            log.Debugf("peer %v not witness enabled, skipping", peer)
            continue
        }

        // Remove sync candidate peers that are no longer candidates due
        // to passing their latest known block.  NOTE: The < is
        // intentional as opposed to <=.  While technically the peer
        // doesn't have a later block when it's equal, it will likely
        // have one soon so it is a reasonable choice.  It also allows
        // the case where both are at 0 such as during regression test.
        if peer.LastBlock() < best.Height {
            state.syncCandidate = false
            continue
        }

        // TODO(davec): Use a better algorithm to choose the best peer.
        // For now, just pick the first available candidate.
        bestPeer = peer
    }

    // Start syncing from the best peer if one was selected.
    if bestPeer != nil {
        // Clear the requestedBlocks if the sync peer changes, otherwise
        // we may ignore blocks we need that the last sync peer failed
        // to send.
        sm.requestedBlocks = make(map[chainhash.Hash]struct{})

        locator, err := sm.chain.LatestBlockLocator()
        if err != nil {
            log.Errorf("Failed to get block locator for the "+
                "latest block: %v", err)
            return
        }

        log.Infof("Syncing to block height %d from peer %v",
            bestPeer.LastBlock(), bestPeer.Addr())

        // When the current height is less than a known checkpoint we
        // can use block headers to learn about which blocks comprise
        // the chain up to the checkpoint and perform less validation
        // for them.  This is possible since each header contains the
        // hash of the previous header and a merkle root.  Therefore if
        // we validate all of the received headers link together
        // properly and the checkpoint hashes match, we can be sure the
        // hashes for the blocks in between are accurate.  Further, once
        // the full blocks are downloaded, the merkle root is computed
        // and compared against the value in the header which proves the
        // full block hasn't been tampered with.
        //
        // Once we have passed the final checkpoint, or checkpoints are
        // disabled, use standard inv messages learn about the blocks
        // and fully validate them.  Finally, regression test mode does
        // not support the headers-first approach so do normal block
        // downloads when in regression test mode.
        if sm.nextCheckpoint != nil &&
            best.Height < sm.nextCheckpoint.Height &&
            sm.chainParams != &chaincfg.RegressionNetParams {

            bestPeer.PushGetHeadersMsg(locator, sm.nextCheckpoint.Hash)
            sm.headersFirstMode = true
            log.Infof("Downloading headers for blocks %d to "+
                "%d from peer %s", best.Height+1,
                sm.nextCheckpoint.Height, bestPeer.Addr())
        } else {
            bestPeer.PushGetBlocksMsg(locator, &zeroHash)
        }
        sm.syncPeer = bestPeer
    } else {
        log.Warnf("No sync peer candidates available")
    }
}
```

>上面的代码大致逻辑:

1. 选择一个bestPeer
2. 清空sm.requestedBlocks
3. 得到最长链的完整locator
4. 调用peer方法，下载区块头，从locator到nextCheckpoint。  
    - **nextCheckpoint会在New时初始化。**
5. sm.headersFirstMode = true  
    - **headersFirstMode是个重要的参数,它表示同步是否完成**

>其中locator，我们看看它的结构：

```go
// BlockLocator is used to help locate a specific block.  The algorithm for
// building the block locator is to add the hashes in reverse order until
// the genesis block is reached.  In order to keep the list of locator hashes
// to a reasonable number of entries, first the most recent previous 12 block
// hashes are added, then the step is doubled each loop iteration to
// exponentially decrease the number of hashes as a function of the distance
// from the block being located.
//
// For example, assume a block chain with a side chain as depicted below:
// 	genesis -> 1 -> 2 -> ... -> 15 -> 16  -> 17  -> 18
// 	                              \-> 16a -> 17a
//
// The block locator for block 17a would be the hashes of blocks:
// [17a 16a 15 14 13 12 11 10 9 8 7 6 4 genesis]
type BlockLocator []*chainhash.Hash
```

#### 1.3.2.1. 请求区块头

>PushGetBlocksMsg

```go

// PushGetBlocksMsg sends a getblocks message for the provided block locator
// and stop hash.  It will ignore back-to-back duplicate requests.
//
// This function is safe for concurrent access.
func (p *Peer) PushGetBlocksMsg(locator blockchain.BlockLocator, stopHash *chainhash.Hash) error {
    // Extract the begin hash from the block locator, if one was specified,
    // to use for filtering duplicate getblocks requests.
    var beginHash *chainhash.Hash
    if len(locator) > 0 {
        beginHash = locator[0]
    }

    // Filter duplicate getblocks requests.
    p.prevGetBlocksMtx.Lock()
    isDuplicate := p.prevGetBlocksStop != nil && p.prevGetBlocksBegin != nil &&
        beginHash != nil && stopHash.IsEqual(p.prevGetBlocksStop) &&
        beginHash.IsEqual(p.prevGetBlocksBegin)
    p.prevGetBlocksMtx.Unlock()

    if isDuplicate {
        log.Tracef("Filtering duplicate [getblocks] with begin "+
            "hash %v, stop hash %v", beginHash, stopHash)
        return nil
    }

    // Construct the getblocks request and queue it to be sent.
    msg := wire.NewMsgGetBlocks(stopHash)
    for _, hash := range locator {
        err := msg.AddBlockLocatorHash(hash)
        if err != nil {
            return err
        }
    }
    p.QueueMessage(msg, nil)

    // Update the previous getblocks request information for filtering
    // duplicates.
    p.prevGetBlocksMtx.Lock()
    p.prevGetBlocksBegin = beginHash
    p.prevGetBlocksStop = stopHash
    p.prevGetBlocksMtx.Unlock()
    return nil
}
```

1. isDuplicate 用于判断重复请求
2. 创建MsgGetBlocks
3. 发送消息
4. 标记prevGetBlocksBegin和prevGetBlocksStop，用于下次判断isDuplicate

#### 1.3.2.2. 接收区块头

当远程节点收到GetHeaders请求时，处理完成之后会返回一个Headers消息，在这里我只把这个处理的代码放上来，先不管它怎么处理。我们先看看收到区块头相关的数据之后，SyncManager是如何处理的。

```go
// OnGetHeaders is invoked when a peer receives a getheaders bitcoin
// message.
func (sp *serverPeer) OnGetHeaders(_ *peer.Peer, msg *wire.MsgGetHeaders) {
    // Ignore getheaders requests if not in sync.
    if !sp.server.syncManager.IsCurrent() {
        return
    }
    chain := sp.server.chain
    headers := chain.LocateHeaders(msg.BlockLocatorHashes, &msg.HashStop)

    // Send found headers to the requesting peer.
    blockHeaders := make([]*wire.BlockHeader, len(headers))
    for i := range headers {
        blockHeaders[i] = &headers[i]
    }
    sp.QueueMessage(&wire.MsgHeaders{Headers: blockHeaders}, nil)
}
```

>此时，远程节点已经响应了请求，并把要求的区块头数据发送过来了。

```go
// OnHeaders is invoked when a peer receives a headers bitcoin
// message.  The message is passed down to the sync manager.
func (sp *serverPeer) OnHeaders(_ *peer.Peer, msg *wire.MsgHeaders) {
    sp.server.syncManager.QueueHeaders(msg, sp.Peer)
}

// QueueHeaders adds the passed headers message and peer to the block handling
// queue.
func (sm *SyncManager) QueueHeaders(headers *wire.MsgHeaders, peer *peerpkg.Peer) {
    // No channel handling here because peers do not need to block on
    // headers messages.
    if atomic.LoadInt32(&sm.shutdown) != 0 {
        return
    }

    sm.msgChan <- &headersMsg{headers: headers, peer: peer}
}
```

写到这个msgChan的数据最后会调用对应的处理方法，头处理如下：

```go
// handleHeadersMsg handles block header messages from all peers.  Headers are
// requested when performing a headers-first sync.
func (sm *SyncManager) handleHeadersMsg(hmsg *headersMsg) {
    peer := hmsg.peer
    _, exists := sm.peerStates[peer]
    if !exists {
        log.Warnf("Received headers message from unknown peer %s", peer)
        return
    }

    // The remote peer is misbehaving if we didn't request headers.
    msg := hmsg.headers
    numHeaders := len(msg.Headers)
    if !sm.headersFirstMode {
        log.Warnf("Got %d unrequested headers from %s -- "+
            "disconnecting", numHeaders, peer.Addr())
        peer.Disconnect()
        return
    }

    // Nothing to do for an empty headers message.
    if numHeaders == 0 {
        return
    }

    // Process all of the received headers ensuring each one connects to the
    // previous and that checkpoints match.
    receivedCheckpoint := false
    var finalHash *chainhash.Hash
    for _, blockHeader := range msg.Headers {
        blockHash := blockHeader.BlockHash()
        finalHash = &blockHash

        // Ensure there is a previous header to compare against.
        prevNodeEl := sm.headerList.Back()
        if prevNodeEl == nil {
            log.Warnf("Header list does not contain a previous" +
                "element as expected -- disconnecting peer")
            peer.Disconnect()
            return
        }

        // Ensure the header properly connects to the previous one and
        // add it to the list of headers.
        node := headerNode{hash: &blockHash}
        prevNode := prevNodeEl.Value.(*headerNode)
        if prevNode.hash.IsEqual(&blockHeader.PrevBlock) {
            node.height = prevNode.height + 1
            e := sm.headerList.PushBack(&node)
            if sm.startHeader == nil {
                sm.startHeader = e
            }
        } else {
            log.Warnf("Received block header that does not "+
                "properly connect to the chain from peer %s "+
                "-- disconnecting", peer.Addr())
            peer.Disconnect()
            return
        }

        // Verify the header at the next checkpoint height matches.
        if node.height == sm.nextCheckpoint.Height {
            if node.hash.IsEqual(sm.nextCheckpoint.Hash) {
                receivedCheckpoint = true
                log.Infof("Verified downloaded block "+
                    "header against checkpoint at height "+
                    "%d/hash %s", node.height, node.hash)
            } else {
                log.Warnf("Block header at height %d/hash "+
                    "%s from peer %s does NOT match "+
                    "expected checkpoint hash of %s -- "+
                    "disconnecting", node.height,
                    node.hash, peer.Addr(),
                    sm.nextCheckpoint.Hash)
                peer.Disconnect()
                return
            }
            break
        }
    }

    // When this header is a checkpoint, switch to fetching the blocks for
    // all of the headers since the last checkpoint.
    if receivedCheckpoint {
        // Since the first entry of the list is always the final block
        // that is already in the database and is only used to ensure
        // the next header links properly, it must be removed before
        // fetching the blocks.
        sm.headerList.Remove(sm.headerList.Front())
        log.Infof("Received %v block headers: Fetching blocks",
            sm.headerList.Len())
        sm.progressLogger.SetLastLogTime(time.Now())
        sm.fetchHeaderBlocks()
        return
    }

    // This header is not a checkpoint, so request the next batch of
    // headers starting from the latest known header and ending with the
    // next checkpoint.
    locator := blockchain.BlockLocator([]*chainhash.Hash{finalHash})
    err := peer.PushGetHeadersMsg(locator, sm.nextCheckpoint.Hash)
    if err != nil {
        log.Warnf("Failed to send getheaders message to "+
            "peer %s: %v", peer.Addr(), err)
        return
    }
}
```

>处理流程：

1. 将消息中的headers处理之后放到headerList
2. 检查是否到了nextCheckpoint
3. 如果到了nextCheckpoint
    1. 检查hash是否正确前做出相对处理
    2. 开始下载区块体数据
4. 否则继续加载头信息

#### 1.3.2.3. 请求区块

```go
// fetchHeaderBlocks creates and sends a request to the syncPeer for the next
// list of blocks to be downloaded based on the current list of headers.
func (sm *SyncManager) fetchHeaderBlocks() {
    // Nothing to do if there is no start header.
    if sm.startHeader == nil {
        log.Warnf("fetchHeaderBlocks called with no start header")
        return
    }

    // Build up a getdata request for the list of blocks the headers
    // describe.  The size hint will be limited to wire.MaxInvPerMsg by
    // the function, so no need to double check it here.
    gdmsg := wire.NewMsgGetDataSizeHint(uint(sm.headerList.Len()))
    numRequested := 0
    for e := sm.startHeader; e != nil; e = e.Next() {
        node, ok := e.Value.(*headerNode)
        if !ok {
            log.Warn("Header list node type is not a headerNode")
            continue
        }

        iv := wire.NewInvVect(wire.InvTypeBlock, node.hash)
        haveInv, err := sm.haveInventory(iv)
        if err != nil {
            log.Warnf("Unexpected failure when checking for "+
                "existing inventory during header block "+
                "fetch: %v", err)
        }
        if !haveInv {
            syncPeerState := sm.peerStates[sm.syncPeer]

            sm.requestedBlocks[*node.hash] = struct{}{}
            syncPeerState.requestedBlocks[*node.hash] = struct{}{}

            // If we're fetching from a witness enabled peer
            // post-fork, then ensure that we receive all the
            // witness data in the blocks.
            if sm.syncPeer.IsWitnessEnabled() {
                iv.Type = wire.InvTypeWitnessBlock
            }

            gdmsg.AddInvVect(iv)
            numRequested++
        }
        sm.startHeader = e.Next()
        if numRequested >= wire.MaxInvPerMsg {
            break
        }
    }
    if len(gdmsg.InvList) > 0 {
        sm.syncPeer.QueueMessage(gdmsg, nil)
    }
}
```

这里会生成一个MsgGetData消息，这个消息体内是一个InvVect列表。在循环体里生成InvVect前，会调用haveInventory检查下这个区块是否已经存在。如果存在就不添加到消息体中。  
**当for结束之前，如果请求的区块数量达到最大值，就会跳出去。否则，sm.startHeader会被设置为空。后面的区块处理中会根据startHeader判断是否再次调用fetchHeaderBlocks()**

#### 1.3.2.4. 接收区块

>当服务器节点收到GetData请求之后，它会发送区块数据过来。OnGetData代码大致如下。

```go
func (sp *serverPeer) OnGetData(_ *peer.Peer, msg *wire.MsgGetData) {
    numAdded := 0
    notFound := wire.NewMsgNotFound()

    length := len(msg.InvList)

    sp.addBanScore(0, uint32(length)*99/wire.MaxInvPerMsg, "getdata")
    var waitChan chan struct{}
    doneChan := make(chan struct{}, 1)

    for i, iv := range msg.InvList {
        var c chan struct{}
        // If this will be the last message we send.
        if i == length-1 && len(notFound.InvList) == 0 {
            c = doneChan
        } else if (i+1)%3 == 0 {
            // Buffered so as to not make the send goroutine block.
            c = make(chan struct{}, 1)
        }
        var err error
        switch iv.Type {
        ...
        case wire.InvTypeBlock:
            err = sp.server.pushBlockMsg(sp, &iv.Hash, c, waitChan, wire.BaseEncoding)
        ...
        default:
            peerLog.Warnf("Unknown type in inventory request %d",
                iv.Type)
            continue
        }
        ...
        numAdded++
        waitChan = c
    }
    if len(notFound.InvList) != 0 {
        sp.QueueMessage(notFound, doneChan)
    }
    if numAdded > 0 {
        <-doneChan
    }
}
```

可以看到它会一个区块发一条消息，回复给请求的节点。由于上面我们请求的是InvTypeBlock类型，因此我把其它类型的处理先删除。如果其中一些区块在本地没有查到，最后，它会发送一个notFound消息，这个消息体中包括了所有未找到的区块信息。

> 回到请求节点逻辑上来，假设这时有收到区块数据。我们看看它是如何处理的。

```go
// OnBlock is invoked when a peer receives a block bitcoin message.  It
// blocks until the bitcoin block has been fully processed.
func (sp *serverPeer) OnBlock(_ *peer.Peer, msg *wire.MsgBlock, buf []byte) {
    // Convert the raw MsgBlock to a btcutil.Block which provides some
    // convenience methods and things such as hash caching.
    block := btcutil.NewBlockFromBlockAndBytes(msg, buf)

    // Add the block to the known inventory for the peer.
    iv := wire.NewInvVect(wire.InvTypeBlock, block.Hash())
    sp.AddKnownInventory(iv)

    // Queue the block up to be handled by the block
    // manager and intentionally block further receives
    // until the bitcoin block is fully processed and known
    // good or bad.  This helps prevent a malicious peer
    // from queuing up a bunch of bad blocks before
    // disconnecting (or being disconnected) and wasting
    // memory.  Additionally, this behavior is depended on
    // by at least the block acceptance test tool as the
    // reference implementation processes blocks in the same
    // thread and therefore blocks further messages until
    // the bitcoin block has been fully processed.
    sp.server.syncManager.QueueBlock(block, sp.Peer, sp.blockProcessed)
    <-sp.blockProcessed
}

// QueueBlock adds the passed block message and peer to the block handling
// queue. Responds to the done channel argument after the block message is
// processed.
func (sm *SyncManager) QueueBlock(block *btcutil.Block, peer *peerpkg.Peer, done chan struct{}) {
    // Don't accept more blocks if we're shutting down.
    if atomic.LoadInt32(&sm.shutdown) != 0 {
        done <- struct{}{}
        return
    }

    sm.msgChan <- &blockMsg{block: block, peer: peer, reply: done}
}
```

收到区块之后，会有大量的处理工作。


#### 1.3.2.5. 处理收到的区块

```go
// handleBlockMsg handles block messages from all peers.
func (sm *SyncManager) handleBlockMsg(bmsg *blockMsg) {
    peer := bmsg.peer
    state, exists := sm.peerStates[peer]
    if !exists {
        log.Warnf("Received block message from unknown peer %s", peer)
        return
    }

    // If we didn't ask for this block then the peer is misbehaving.
    blockHash := bmsg.block.Hash()
    if _, exists = state.requestedBlocks[*blockHash]; !exists {
        // The regression test intentionally sends some blocks twice
        // to test duplicate block insertion fails.  Don't disconnect
        // the peer or ignore the block when we're in regression test
        // mode in this case so the chain code is actually fed the
        // duplicate blocks.
        if sm.chainParams != &chaincfg.RegressionNetParams {
            log.Warnf("Got unrequested block %v from %s -- "+
                "disconnecting", blockHash, peer.Addr())
            peer.Disconnect()
            return
        }
    }

    // When in headers-first mode, if the block matches the hash of the
    // first header in the list of headers that are being fetched, it's
    // eligible for less validation since the headers have already been
    // verified to link together and are valid up to the next checkpoint.
    // Also, remove the list entry for all blocks except the checkpoint
    // since it is needed to verify the next round of headers links
    // properly.
    isCheckpointBlock := false
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

    // Remove block from request maps. Either chain will know about it and
    // so we shouldn't have any more instances of trying to fetch it, or we
    // will fail the insert and thus we'll retry next time we get an inv.
    delete(state.requestedBlocks, *blockHash)
    delete(sm.requestedBlocks, *blockHash)

    // Process the block to include validation, best chain selection, orphan
    // handling, etc.
    _, isOrphan, err := sm.chain.ProcessBlock(bmsg.block, behaviorFlags)
    if err != nil {
        // When the error is a rule error, it means the block was simply
        // rejected as opposed to something actually going wrong, so log
        // it as such.  Otherwise, something really did go wrong, so log
        // it as an actual error.
        if _, ok := err.(blockchain.RuleError); ok {
            log.Infof("Rejected block %v from %s: %v", blockHash,
                peer, err)
        } else {
            log.Errorf("Failed to process block %v: %v",
                blockHash, err)
        }
        if dbErr, ok := err.(database.Error); ok && dbErr.ErrorCode ==
            database.ErrCorruption {
            panic(dbErr)
        }

        // Convert the error into an appropriate reject message and
        // send it.
        code, reason := mempool.ErrToRejectErr(err)
        peer.PushRejectMsg(wire.CmdBlock, code, reason, blockHash, false)
        return
    }

    // Meta-data about the new block this peer is reporting. We use this
    // below to update this peer's lastest block height and the heights of
    // other peers based on their last announced block hash. This allows us
    // to dynamically update the block heights of peers, avoiding stale
    // heights when looking for a new sync peer. Upon acceptance of a block
    // or recognition of an orphan, we also use this information to update
    // the block heights over other peers who's invs may have been ignored
    // if we are actively syncing while the chain is not yet current or
    // who may have lost the lock announcment race.
    var heightUpdate int32
    var blkHashUpdate *chainhash.Hash

    // Request the parents for the orphan block from the peer that sent it.
    if isOrphan {
        // We've just received an orphan block from a peer. In order
        // to update the height of the peer, we try to extract the
        // block height from the scriptSig of the coinbase transaction.
        // Extraction is only attempted if the block's version is
        // high enough (ver 2+).
        header := &bmsg.block.MsgBlock().Header
        if blockchain.ShouldHaveSerializedBlockHeight(header) {
            coinbaseTx := bmsg.block.Transactions()[0]
            cbHeight, err := blockchain.ExtractCoinbaseHeight(coinbaseTx)
            if err != nil {
                log.Warnf("Unable to extract height from "+
                    "coinbase tx: %v", err)
            } else {
                log.Debugf("Extracted height of %v from "+
                    "orphan block", cbHeight)
                heightUpdate = cbHeight
                blkHashUpdate = blockHash
            }
        }

        orphanRoot := sm.chain.GetOrphanRoot(blockHash)
        locator, err := sm.chain.LatestBlockLocator()
        if err != nil {
            log.Warnf("Failed to get block locator for the "+
                "latest block: %v", err)
        } else {
            peer.PushGetBlocksMsg(locator, orphanRoot)
        }
    } else {
        // When the block is not an orphan, log information about it and
        // update the chain state.
        sm.progressLogger.LogBlockHeight(bmsg.block)

        // Update this peer's latest block height, for future
        // potential sync node candidacy.
        best := sm.chain.BestSnapshot()
        heightUpdate = best.Height
        blkHashUpdate = &best.Hash

        // Clear the rejected transactions.
        sm.rejectedTxns = make(map[chainhash.Hash]struct{})
    }

    // Update the block height for this peer. But only send a message to
    // the server for updating peer heights if this is an orphan or our
    // chain is "current". This avoids sending a spammy amount of messages
    // if we're syncing the chain from scratch.
    if blkHashUpdate != nil && heightUpdate != 0 {
        peer.UpdateLastBlockHeight(heightUpdate)
        if isOrphan || sm.current() {
            go sm.peerNotifier.UpdatePeerHeights(blkHashUpdate, heightUpdate,
                peer)
        }
    }

    // Nothing more to do if we aren't in headers-first mode.
    if !sm.headersFirstMode {
        return
    }

    // This is headers-first mode, so if the block is not a checkpoint
    // request more blocks using the header list when the request queue is
    // getting short.
    if !isCheckpointBlock {
        if sm.startHeader != nil &&
            len(state.requestedBlocks) < minInFlightBlocks {
            sm.fetchHeaderBlocks()
        }
        return
    }

    // This is headers-first mode and the block is a checkpoint.  When
    // there is a next checkpoint, get the next round of headers by asking
    // for headers starting from the block after this one up to the next
    // checkpoint.
    prevHeight := sm.nextCheckpoint.Height
    prevHash := sm.nextCheckpoint.Hash
    sm.nextCheckpoint = sm.findNextHeaderCheckpoint(prevHeight)
    if sm.nextCheckpoint != nil {
        locator := blockchain.BlockLocator([]*chainhash.Hash{prevHash})
        err := peer.PushGetHeadersMsg(locator, sm.nextCheckpoint.Hash)
        if err != nil {
            log.Warnf("Failed to send getheaders message to "+
                "peer %s: %v", peer.Addr(), err)
            return
        }
        log.Infof("Downloading headers for blocks %d to %d from "+
            "peer %s", prevHeight+1, sm.nextCheckpoint.Height,
            sm.syncPeer.Addr())
        return
    }

    // This is headers-first mode, the block is a checkpoint, and there are
    // no more checkpoints, so switch to normal mode by requesting blocks
    // from the block after this one up to the end of the chain (zero hash).
    sm.headersFirstMode = false
    sm.headerList.Init()
    log.Infof("Reached the final checkpoint -- switching to normal mode")
    locator := blockchain.BlockLocator([]*chainhash.Hash{blockHash})
    err = peer.PushGetBlocksMsg(locator, &zeroHash)
    if err != nil {
        log.Warnf("Failed to send getblocks message to peer %s: %v",
            peer.Addr(), err)
        return
    }
}
```

> **处理流程：**

1. 基本检查
2. 判断isCheckpointBlock  
    **由于在处理区块头时，当receivedCheckpoint=true时，headerList会把头元素删除。因此，在这里如果没有达到CheckpointBlock时，会删除headerList头元素。否则就说明headerList已经为空了。**
3. 处理区块
   **处理完成之后，会返回：是否在主链和是否为孤儿节点(也就是找不到父亲节点)。如果是孤儿节点，会添加的区块链的orphans中**
4. 如果isOrphan=true。
    1. 从coinbaseTx交易中得到区块高
    2. 得到孤儿根节点hash。用PushGetBlocksMsg去请求空缺的区块。
5. 如果headersFirstMode=false，结束。（在同步完成之前，这个参数为false）
6. 如果isCheckpointBlock=false
    1. 如果前面发送GetData请求时，startHeader没有设为空，这里继续fetchHeaderBlocks()
    2. 如果startHeader为空，直接返回
7. 重置sm.nextCheckpoint,如果未找到，就为空
8. 如果到了CheckpointBlock，就得到下一个CheckpointBlock。如果这个CheckpointBlock存在，说明同步还没有完成。此时会调用GetHeaders消息继续。
9. 最后，没有nextCheckpoint之后设置headersFirstMode=false.直接调用PushGetBlocksMsg请求同步到最新的区块（设置第二个参数为zeroHash）。

### 1.3.3. MsgGetHeaders处理

当一个节点收到其它节点的MsgGetHeaders请求时，它会从自己的本地读取区块头节点返回。

```go
// OnGetHeaders is invoked when a peer receives a getheaders bitcoin
// message.
func (sp *serverPeer) OnGetHeaders(_ *peer.Peer, msg *wire.MsgGetHeaders) {
    // Ignore getheaders requests if not in sync.
    if !sp.server.syncManager.IsCurrent() {
        return
    }

    // Find the most recent known block in the best chain based on the block
    // locator and fetch all of the headers after it until either
    // wire.MaxBlockHeadersPerMsg have been fetched or the provided stop
    // hash is encountered.
    //
    // Use the block after the genesis block if no other blocks in the
    // provided locator are known.  This does mean the client will start
    // over with the genesis block if unknown block locators are provided.
    //
    // This mirrors the behavior in the reference implementation.
    chain := sp.server.chain
    headers := chain.LocateHeaders(msg.BlockLocatorHashes, &msg.HashStop)

    // Send found headers to the requesting peer.
    blockHeaders := make([]*wire.BlockHeader, len(headers))
    for i := range headers {
        blockHeaders[i] = &headers[i]
    }
    sp.QueueMessage(&wire.MsgHeaders{Headers: blockHeaders}, nil)
}
```

#### 1.3.3.1. LocateHeaders

```go
// LocateHeaders returns the headers of the blocks after the first known block
// in the locator until the provided stop hash is reached, or up to a max of
// wire.MaxBlockHeadersPerMsg headers.
//
// In addition, there are two special cases:
//
// - When no locators are provided, the stop hash is treated as a request for
//   that header, so it will either return the header for the stop hash itself
//   if it is known, or nil if it is unknown
// - When locators are provided, but none of them are known, headers starting
//   after the genesis block will be returned
//
// This function is safe for concurrent access.
func (b *BlockChain) LocateHeaders(locator BlockLocator, hashStop *chainhash.Hash) []wire.BlockHeader {
    b.chainLock.RLock()
    headers := b.locateHeaders(locator, hashStop, wire.MaxBlockHeadersPerMsg)
    b.chainLock.RUnlock()
    return headers
}


// locateHeaders returns the headers of the blocks after the first known block
// in the locator until the provided stop hash is reached, or up to the provided
// max number of block headers.
//
// See the comment on the exported function for more details on special cases.
//
// This function MUST be called with the chain state lock held (for reads).
func (b *BlockChain) locateHeaders(locator BlockLocator, hashStop *chainhash.Hash, maxHeaders uint32) []wire.BlockHeader {
    // Find the node after the first known block in the locator and the
    // total number of nodes after it needed while respecting the stop hash
    // and max entries.
    node, total := b.locateInventory(locator, hashStop, maxHeaders)
    if total == 0 {
        return nil
    }

    // Populate and return the found headers.
    headers := make([]wire.BlockHeader, 0, total)
    for i := uint32(0); i < total; i++ {
        headers = append(headers, node.Header())
        node = b.bestChain.Next(node)
    }
    return headers
}
```

>1. locateInventory 用于从blockIndex中读取locator中的启初节点和节点总数
>2. bestChain结构为chainView，chainView是维护在内存中的对象，方便对链中节点各种处理。b.bestChain.Next(node)读取下一个节点。

>locateInventory

```go
// locateInventory returns the node of the block after the first known block in
// the locator along with the number of subsequent nodes needed to either reach
// the provided stop hash or the provided max number of entries.
//
// In addition, there are two special cases:
//
// - When no locators are provided, the stop hash is treated as a request for
//   that block, so it will either return the node associated with the stop hash
//   if it is known, or nil if it is unknown
// - When locators are provided, but none of them are known, nodes starting
//   after the genesis block will be returned
//
// This is primarily a helper function for the locateBlocks and locateHeaders
// functions.
//
// This function MUST be called with the chain state lock held (for reads).
func (b *BlockChain) locateInventory(locator BlockLocator, hashStop *chainhash.Hash, maxEntries uint32) (*blockNode, uint32) {
    // There are no block locators so a specific block is being requested
    // as identified by the stop hash.
    stopNode := b.index.LookupNode(hashStop)
    if len(locator) == 0 {
        if stopNode == nil {
            // No blocks with the stop hash were found so there is
            // nothing to do.
            return nil, 0
        }
        return stopNode, 1
    }

    // Find the most recent locator block hash in the main chain.  In the
    // case none of the hashes in the locator are in the main chain, fall
    // back to the genesis block.
    startNode := b.bestChain.Genesis()
    for _, hash := range locator {
        node := b.index.LookupNode(hash)
        if node != nil && b.bestChain.Contains(node) {
            startNode = node
            break
        }
    }

    // Start at the block after the most recently known block.  When there
    // is no next block it means the most recently known block is the tip of
    // the best chain, so there is nothing more to do.
    startNode = b.bestChain.Next(startNode)
    if startNode == nil {
        return nil, 0
    }

    // Calculate how many entries are needed.
    total := uint32((b.bestChain.Tip().height - startNode.height) + 1)
    if stopNode != nil && b.bestChain.Contains(stopNode) &&
        stopNode.height >= startNode.height {

        total = uint32((stopNode.height - startNode.height) + 1)
    }
    if total > maxEntries {
        total = maxEntries
    }

    return startNode, total
}
```

> chainView结构：

```go
// chainView provides a flat view of a specific branch of the block chain from
// its tip back to the genesis block and provides various convenience functions
// for comparing chains.
//
// For example, assume a block chain with a side chain as depicted below:
//   genesis -> 1 -> 2 -> 3 -> 4  -> 5 ->  6  -> 7  -> 8
//                         \-> 4a -> 5a -> 6a
//
// The chain view for the branch ending in 6a consists of:
//   genesis -> 1 -> 2 -> 3 -> 4a -> 5a -> 6a
type chainView struct {
    mtx   sync.Mutex
    nodes []*blockNode
}

func (c *chainView) next(node *blockNode) *blockNode {
    if node == nil || !c.contains(node) {
        return nil
    }

    return c.nodeByHeight(node.height + 1)
}

func (c *chainView) nodeByHeight(height int32) *blockNode {
    if height < 0 || height >= int32(len(c.nodes)) {
        return nil
    }

    return c.nodes[height]
}
```

### 1.3.4. MsgGetData请求处理

OnGetData是个复合的接口，可以处理各种InvType类型的请求。

```go

// InvType represents the allowed types of inventory vectors.  See InvVect.
type InvType uint32

// These constants define the various supported inventory vector types.
const (
    InvTypeError                InvType = 0
    InvTypeTx                   InvType = 1
    InvTypeBlock                InvType = 2
    InvTypeFilteredBlock        InvType = 3
    InvTypeWitnessBlock         InvType = InvTypeBlock | InvWitnessFlag
    InvTypeWitnessTx            InvType = InvTypeTx | InvWitnessFlag
    InvTypeFilteredWitnessBlock InvType = InvTypeFilteredBlock | InvWitnessFlag
)
```

>OnGetData

```go
// handleGetData is invoked when a peer receives a getdata bitcoin message and
// is used to deliver block and transaction information.
func (sp *serverPeer) OnGetData(_ *peer.Peer, msg *wire.MsgGetData) {
    numAdded := 0
    notFound := wire.NewMsgNotFound()

    length := len(msg.InvList)
    // A decaying ban score increase is applied to prevent exhausting resources
    // with unusually large inventory queries.
    // Requesting more than the maximum inventory vector length within a short
    // period of time yields a score above the default ban threshold. Sustained
    // bursts of small requests are not penalized as that would potentially ban
    // peers performing IBD.
    // This incremental score decays each minute to half of its value.
    sp.addBanScore(0, uint32(length)*99/wire.MaxInvPerMsg, "getdata")

    // We wait on this wait channel periodically to prevent queuing
    // far more data than we can send in a reasonable time, wasting memory.
    // The waiting occurs after the database fetch for the next one to
    // provide a little pipelining.
    var waitChan chan struct{}
    doneChan := make(chan struct{}, 1)

    for i, iv := range msg.InvList {
        var c chan struct{}
        // If this will be the last message we send.
        if i == length-1 && len(notFound.InvList) == 0 {
            c = doneChan
        } else if (i+1)%3 == 0 {
            // Buffered so as to not make the send goroutine block.
            c = make(chan struct{}, 1)
        }
        var err error
        switch iv.Type {
        case wire.InvTypeWitnessTx:
            err = sp.server.pushTxMsg(sp, &iv.Hash, c, waitChan, wire.WitnessEncoding)
        case wire.InvTypeTx:
            err = sp.server.pushTxMsg(sp, &iv.Hash, c, waitChan, wire.BaseEncoding)
        case wire.InvTypeWitnessBlock:
            err = sp.server.pushBlockMsg(sp, &iv.Hash, c, waitChan, wire.WitnessEncoding)
        case wire.InvTypeBlock:
            err = sp.server.pushBlockMsg(sp, &iv.Hash, c, waitChan, wire.BaseEncoding)
        case wire.InvTypeFilteredWitnessBlock:
            err = sp.server.pushMerkleBlockMsg(sp, &iv.Hash, c, waitChan, wire.WitnessEncoding)
        case wire.InvTypeFilteredBlock:
            err = sp.server.pushMerkleBlockMsg(sp, &iv.Hash, c, waitChan, wire.BaseEncoding)
        default:
            peerLog.Warnf("Unknown type in inventory request %d",
                iv.Type)
            continue
        }
        if err != nil {
            notFound.AddInvVect(iv)

            // When there is a failure fetching the final entry
            // and the done channel was sent in due to there
            // being no outstanding not found inventory, consume
            // it here because there is now not found inventory
            // that will use the channel momentarily.
            if i == len(msg.InvList)-1 && c != nil {
                <-c
            }
        }
        numAdded++
        waitChan = c
    }
    if len(notFound.InvList) != 0 {
        sp.QueueMessage(notFound, doneChan)
    }

    // Wait for messages to be sent. We can send quite a lot of data at this
    // point and this will keep the peer busy for a decent amount of time.
    // We don't process anything else by them in this time so that we
    // have an idea of when we should hear back from them - else the idle
    // timeout could fire when we were only half done sending the blocks.
    if numAdded > 0 {
        <-doneChan
    }
}
```

这个方法在上面提过。我们主要看下pushBlockMsg方法中的处理。其它的InvType先不管。

```go
// pushBlockMsg sends a block message for the provided block hash to the
// connected peer.  An error is returned if the block hash is not known.
func (s *server) pushBlockMsg(sp *serverPeer, hash *chainhash.Hash, doneChan chan<- struct{},
    waitChan <-chan struct{}, encoding wire.MessageEncoding) error {

    // Fetch the raw block bytes from the database.
    var blockBytes []byte
    err := sp.server.db.View(func(dbTx database.Tx) error {
        var err error
        blockBytes, err = dbTx.FetchBlock(hash)
        return err
    })
    if err != nil {
        peerLog.Tracef("Unable to fetch requested block hash %v: %v",
            hash, err)

        if doneChan != nil {
            doneChan <- struct{}{}
        }
        return err
    }

    // Deserialize the block.
    var msgBlock wire.MsgBlock
    err = msgBlock.Deserialize(bytes.NewReader(blockBytes))
    if err != nil {
        peerLog.Tracef("Unable to deserialize requested block hash "+
            "%v: %v", hash, err)

        if doneChan != nil {
            doneChan <- struct{}{}
        }
        return err
    }

    // Once we have fetched data wait for any previous operation to finish.
    if waitChan != nil {
        <-waitChan
    }

    // We only send the channel for this message if we aren't sending
    // an inv straight after.
    var dc chan<- struct{}
    continueHash := sp.continueHash
    sendInv := continueHash != nil && continueHash.IsEqual(hash)
    if !sendInv {
        dc = doneChan
    }
    sp.QueueMessageWithEncoding(&msgBlock, dc, encoding)

    // When the peer requests the final block that was advertised in
    // response to a getblocks message which requested more blocks than
    // would fit into a single message, send it a new inventory message
    // to trigger it to issue another getblocks message for the next
    // batch of inventory.
    if sendInv {
        best := sp.server.chain.BestSnapshot()
        invMsg := wire.NewMsgInvSizeHint(1)
        iv := wire.NewInvVect(wire.InvTypeBlock, &best.Hash)
        invMsg.AddInvVect(iv)
        sp.QueueMessage(invMsg, doneChan)
        sp.continueHash = nil
    }
    return nil
}
```

1. 读取block数据并反序列化为msgBlock对象
2. 发送消息

在第一次调用此方法时，waitChan为空，doneChan也就是外层OnGetData的c，然后这个c会被传入消息发送方法QueueMessageWithEncoding中，只有消息发送完成才会向这个通道写通知，当这个方法调用返回之后，waitChan=c.也就是说当第二次进入这个方法时，如果第一个消息还没有发送完成，在这时<-waitChan会阻塞。如此依次串行。直到最后OnGetData循环完成，c = doneChan，最后一条消息发送完成结束。
**continueHash是在OnGetBlocks处理的。为了分批回复invMsg，所以在最后才会有那段代码，这里可以认为它为false。**

#### 1.3.4.1. FetchBlock

```go
// FetchBlock returns the raw serialized bytes for the block identified by the
// given hash.  The raw bytes are in the format returned by Serialize on a
// wire.MsgBlock.
//
// Returns the following errors as required by the interface contract:
//   - ErrBlockNotFound if the requested block hash does not exist
//   - ErrTxClosed if the transaction has already been closed
//   - ErrCorruption if the database has somehow become corrupted
//
// In addition, returns ErrDriverSpecific if any failures occur when reading the
// block files.
//
// NOTE: The data returned by this function is only valid during a database
// transaction.  Attempting to access it after a transaction has ended results
// in undefined behavior.  This constraint prevents additional data copies and
// allows support for memory-mapped database implementations.
//
// This function is part of the database.Tx interface implementation.
func (tx *transaction) FetchBlock(hash *chainhash.Hash) ([]byte, error) {
    // Ensure transaction state is valid.
    if err := tx.checkClosed(); err != nil {
        return nil, err
    }

    // When the block is pending to be written on commit return the bytes
    // from there.
    if idx, exists := tx.pendingBlocks[*hash]; exists {
        return tx.pendingBlockData[idx].bytes, nil
    }

    // Lookup the location of the block in the files from the block index.
    blockRow, err := tx.fetchBlockRow(hash)
    if err != nil {
        return nil, err
    }
    location := deserializeBlockLoc(blockRow)

    // Read the block from the appropriate location.  The function also
    // performs a checksum over the data to detect data corruption.
    blockBytes, err := tx.db.store.readBlock(hash, location)
    if err != nil {
        return nil, err
    }

    return blockBytes, nil
}
```

> 1. 调用fetchBlockRow从blockIdxBucket中读取blockLocation
> 2. 调用store.readBlock从文件中读取区块数据

## 1.4. 数据传播

同步管理器还有一个重要的功能就是数据的转发。比如收到一个新生成的区块时，它会广播给其它节点。

### 1.4.1. 订阅通知

在New一个SyncManager的方法中，我们看到了会调用接口，订阅通知。

```go
sm.chain.Subscribe(sm.handleBlockchainNotification)
```

进入这个方法，看下它做了什么事。

```go
// handleBlockchainNotification handles notifications from blockchain.  It does
// things such as request orphan block parents and relay accepted blocks to
// connected peers.
func (sm *SyncManager) handleBlockchainNotification(notification *blockchain.Notification) {
    switch notification.Type {
    // A block has been accepted into the block chain.  Relay it to other
    // peers.
    case blockchain.NTBlockAccepted:
        // Don't relay if we are not current. Other peers that are
        // current should already know about it.
        if !sm.current() {
            return
        }

        block, ok := notification.Data.(*btcutil.Block)
        if !ok {
            log.Warnf("Chain accepted notification is not a block.")
            break
        }

        // Generate the inventory vector and relay it.
        iv := wire.NewInvVect(wire.InvTypeBlock, block.Hash())
        sm.peerNotifier.RelayInventory(iv, block.MsgBlock().Header)

    // A block has been connected to the main block chain.
    case blockchain.NTBlockConnected:
        block, ok := notification.Data.(*btcutil.Block)
        if !ok {
            log.Warnf("Chain connected notification is not a block.")
            break
        }

        // Remove all of the transactions (except the coinbase) in the
        // connected block from the transaction pool.  Secondly, remove any
        // transactions which are now double spends as a result of these
        // new transactions.  Finally, remove any transaction that is
        // no longer an orphan. Transactions which depend on a confirmed
        // transaction are NOT removed recursively because they are still
        // valid.
        for _, tx := range block.Transactions()[1:] {
            sm.txMemPool.RemoveTransaction(tx, false)
            sm.txMemPool.RemoveDoubleSpends(tx)
            sm.txMemPool.RemoveOrphan(tx)
            sm.peerNotifier.TransactionConfirmed(tx)
            acceptedTxs := sm.txMemPool.ProcessOrphans(tx)
            sm.peerNotifier.AnnounceNewTransactions(acceptedTxs)
        }

        // Register block with the fee estimator, if it exists.
        if sm.feeEstimator != nil {
            err := sm.feeEstimator.RegisterBlock(block)

            // If an error is somehow generated then the fee estimator
            // has entered an invalid state. Since it doesn't know how
            // to recover, create a new one.
            if err != nil {
                sm.feeEstimator = mempool.NewFeeEstimator(
                    mempool.DefaultEstimateFeeMaxRollback,
                    mempool.DefaultEstimateFeeMinRegisteredBlocks)
            }
        }

    // A block has been disconnected from the main block chain.
    case blockchain.NTBlockDisconnected:
        block, ok := notification.Data.(*btcutil.Block)
        if !ok {
            log.Warnf("Chain disconnected notification is not a block.")
            break
        }

        // Reinsert all of the transactions (except the coinbase) into
        // the transaction pool.
        for _, tx := range block.Transactions()[1:] {
            _, _, err := sm.txMemPool.MaybeAcceptTransaction(tx,
                false, false)
            if err != nil {
                // Remove the transaction and all transactions
                // that depend on it if it wasn't accepted into
                // the transaction pool.
                sm.txMemPool.RemoveTransaction(tx, true)
            }
        }

        // Rollback previous block recorded by the fee estimator.
        if sm.feeEstimator != nil {
            sm.feeEstimator.Rollback(block.Hash())
        }
    }
}
```

>看下三种类型的区别：

```go
// Constants for the type of a notification message.
const (
    // NTBlockAccepted indicates the associated block was accepted into
    // the block chain.  Note that this does not necessarily mean it was
    // added to the main chain.  For that, use NTBlockConnected.
    NTBlockAccepted NotificationType = iota

    // NTBlockConnected indicates the associated block was connected to the
    // main chain.
    NTBlockConnected

    // NTBlockDisconnected indicates the associated block was disconnected
    // from the main chain.
    NTBlockDisconnected
)
```

>- NTBlockAccepted是在BlockChain->maybeAcceptBlock方法中调用的，我们看下它的说明：  
**potentially accepts a block into the block chain and, if accepted, returns whether or not it is on the main chain.**
>- NTBlockConnected是BlockChain->connectBlock方法中调用的，我们看下它的说明：  
**handles connecting the passed node/block to the end of the main (best) chain.**

### 1.4.2. 通知处理

在同步管理器中，它会做一些处理操作，然后把消息传播出去。调用的就是peerNotifier。

>其中PeerNotifier这个接口有四个方法：

```go
// PeerNotifier exposes methods to notify peers of status changes to
// transactions, blocks, etc. Currently server (in the main package) implements
// this interface.
type PeerNotifier interface {
    AnnounceNewTransactions(newTxs []*mempool.TxDesc)

    UpdatePeerHeights(latestBlkHash *chainhash.Hash, latestHeight int32, updateSource *peer.Peer)

    RelayInventory(invVect *wire.InvVect, data interface{})

    TransactionConfirmed(tx *btcutil.Tx)
}
```

因为在创建SyncManager时，server把自己当为这个参数传入过来。所以这些接口的实现都在server中。

#### 1.4.2.1. peerNotifier实现

```go
// RelayInventory relays the passed inventory vector to all connected peers
// that are not already known to have it.
func (s *server) RelayInventory(invVect *wire.InvVect, data interface{}) {
    s.relayInv <- relayMsg{invVect: invVect, data: data}
}

// relayTransactions generates and relays inventory vectors for all of the
// passed transactions to all connected peers.
func (s *server) relayTransactions(txns []*mempool.TxDesc) {
    for _, txD := range txns {
        iv := wire.NewInvVect(wire.InvTypeTx, txD.Tx.Hash())
        s.RelayInventory(iv, txD)
    }
}

// AnnounceNewTransactions generates and relays inventory vectors and notifies
// both websocket and getblocktemplate long poll clients of the passed
// transactions.  This function should be called whenever new transactions
// are added to the mempool.
func (s *server) AnnounceNewTransactions(txns []*mempool.TxDesc) {
    // Generate and relay inventory vectors for all newly accepted
    // transactions.
    s.relayTransactions(txns)

    // Notify both websocket and getblocktemplate long poll clients of all
    // newly accepted transactions.
    if s.rpcServer != nil {
        s.rpcServer.NotifyNewTransactions(txns)
    }
}

// Transaction has one confirmation on the main chain. Now we can mark it as no
// longer needing rebroadcasting.
func (s *server) TransactionConfirmed(tx *btcutil.Tx) {
    // Rebroadcasting is only necessary when the RPC server is active.
    if s.rpcServer == nil {
        return
    }

    iv := wire.NewInvVect(wire.InvTypeTx, tx.Hash())
    s.RemoveRebroadcastInventory(iv)
}
```

### 1.4.3. 库存广播

不管是区块。还是新交易，都会调用到RelayInventory。比如钱包给rpcServer发了一条交易，rpc服务会调用AnnounceNewTransactions。

>**节点层级的处理都在server.peerHandler中；**

```go
func (s *server) peerHandler() {
    ...
    // New inventory to potentially be relayed to other peers.
    case invMsg := <-s.relayInv:
        s.handleRelayInvMsg(state, invMsg)
    ...
}
```

#### 1.4.3.1. handleRelayInvMsg

```go
// handleRelayInvMsg deals with relaying inventory to peers that are not already
// known to have it.  It is invoked from the peerHandler goroutine.
func (s *server) handleRelayInvMsg(state *peerState, msg relayMsg) {
    state.forAllPeers(func(sp *serverPeer) {
        if !sp.Connected() {
            return
        }

        // If the inventory is a block and the peer prefers headers,
        // generate and send a headers message instead of an inventory
        // message.
        if msg.invVect.Type == wire.InvTypeBlock && sp.WantsHeaders() {
            blockHeader, ok := msg.data.(wire.BlockHeader)
            if !ok {
                peerLog.Warnf("Underlying data for headers" +
                    " is not a block header")
                return
            }
            msgHeaders := wire.NewMsgHeaders()
            if err := msgHeaders.AddBlockHeader(&blockHeader); err != nil {
                peerLog.Errorf("Failed to add block"+
                    " header: %v", err)
                return
            }
            sp.QueueMessage(msgHeaders, nil)
            return
        }

        if msg.invVect.Type == wire.InvTypeTx {
            // Don't relay the transaction to the peer when it has
            // transaction relaying disabled.
            if sp.relayTxDisabled() {
                return
            }

            txD, ok := msg.data.(*mempool.TxDesc)
            if !ok {
                peerLog.Warnf("Underlying data for tx inv "+
                    "relay is not a *mempool.TxDesc: %T",
                    msg.data)
                return
            }

            // Don't relay the transaction if the transaction fee-per-kb
            // is less than the peer's feefilter.
            feeFilter := atomic.LoadInt64(&sp.feeFilter)
            if feeFilter > 0 && txD.FeePerKB < feeFilter {
                return
            }

            // Don't relay the transaction if there is a bloom
            // filter loaded and the transaction doesn't match it.
            if sp.filter.IsLoaded() {
                if !sp.filter.MatchTxAndUpdate(txD.Tx) {
                    return
                }
            }
        }

        // Queue the inventory to be relayed with the next batch.
        // It will be ignored if the peer is already known to
        // have the inventory.
        sp.QueueInventory(msg.invVect)
    })
}
```

这个方法会取出所有的节点(进来或者出去的)。
- 如果是区块，并且WantsHeaders，就发送MsgHeaders给对方。前面已经分析过，如果收到这个消息，它没有库存，就会调用GetData去取区块数据。
- 如果是交易，会做些基本的判断。最后一个会有bloom filter的处理，这里先不管。
- 如果上面的逻辑没有返回，就会调用sp.QueueInventory。

>这里进入了peer中的QueueInventory方法：

```go
// QueueInventory adds the passed inventory to the inventory send queue which
// might not be sent right away, rather it is trickled to the peer in batches.
// Inventory that the peer is already known to have is ignored.
//
// This function is safe for concurrent access.
func (p *Peer) QueueInventory(invVect *wire.InvVect) {
    // Don't add the inventory to the send queue if the peer is already
    // known to have it.
    if p.knownInventory.Exists(invVect) {
        return
    }

    // Avoid risk of deadlock if goroutine already exited.  The goroutine
    // we will be sending to hangs around until it knows for a fact that
    // it is marked as disconnected and *then* it drains the channels.
    if !p.Connected() {
        return
    }

    p.outputInvChan <- invVect
}
```

当此节点收到某个peer的invVect时，会缓存起来，因此，在这里会先判断下。

>写到outputInvChan的数据，最后就在出去的queueHandler中处理

```go
func (p *Peer) queueHandler() {
    ...
    // To avoid duplication below.
    queuePacket := func(msg outMsg, list *list.List, waiting bool) bool {
        if !waiting {
            p.sendQueue <- msg
        } else {
            list.PushBack(msg)
        }
        // we are always waiting now.
        return true
    }

    ...
    case iv := <-p.outputInvChan:
        // No handshake?  They'll find out soon enough.
        if p.VersionKnown() {
            // If this is a new block, then we'll blast it
            // out immediately, sipping the inv trickle
            // queue.
            if iv.Type == wire.InvTypeBlock ||
                iv.Type == wire.InvTypeWitnessBlock {

                invMsg := wire.NewMsgInvSizeHint(1)
                invMsg.AddInvVect(iv)
                waiting = queuePacket(outMsg{msg: invMsg},
                    pendingMsgs, waiting)
            } else {
                invSendQueue.PushBack(iv)
            }
        }
    case <-trickleTicker.C:
        // Don't send anything if we're disconnecting or there
        // is no queued inventory.
        // version is known if send queue has any entries.
        if atomic.LoadInt32(&p.disconnect) != 0 ||
            invSendQueue.Len() == 0 {
            continue
        }

        // Create and send as many inv messages as needed to
        // drain the inventory send queue.
        invMsg := wire.NewMsgInvSizeHint(uint(invSendQueue.Len()))
        for e := invSendQueue.Front(); e != nil; e = invSendQueue.Front() {
            iv := invSendQueue.Remove(e).(*wire.InvVect)

            // Don't send inventory that became known after
            // the initial check.
            if p.knownInventory.Exists(iv) {
                continue
            }

            invMsg.AddInvVect(iv)
            if len(invMsg.InvList) >= maxInvTrickleSize {
                waiting = queuePacket(
                    outMsg{msg: invMsg},
                    pendingMsgs, waiting)
                invMsg = wire.NewMsgInvSizeHint(uint(invSendQueue.Len()))
            }

            // Add the inventory that is being relayed to
            // the known inventory for the peer.
            p.AddKnownInventory(iv)
        }
        if len(invMsg.InvList) > 0 {
            waiting = queuePacket(outMsg{msg: invMsg},
                pendingMsgs, waiting)
        }
}
```

如果是InvTypeBlock或者InvTypeWitnessBlock类型，上面也有说明，表示优先级比较高，直接包装成invMsg，放到queuePacket中等待发送。否则就放到invSendQueue中，等待trickleTicker定时处理，它会把在这个时间段内收到的iv缓冲数据一次性发出去。在这个过程中，如果iv数据达到maxInvTrickleSize(1000)，它会马上生成outMsg写到queuePacket中。并且创建一个新的invMsg。

>MsgInv

```go
// MsgInv implements the Message interface and represents a bitcoin inv message.
// It is used to advertise a peer's known data such as blocks and transactions
// through inventory vectors.  It may be sent unsolicited to inform other peers
// of the data or in response to a getblocks message (MsgGetBlocks).  Each
// message is limited to a maximum number of inventory vectors, which is
// currently 50,000.
//
// Use the AddInvVect function to build up the list of inventory vectors when
// sending an inv message to another peer.
type MsgInv struct {
    InvList []*InvVect
}
```

#### 1.4.3.2. OnInv

当远程peer节点收到其它节点发的MsgInv消息。会在OnInv中处理。

```go

// OnInv is invoked when a peer receives an inv bitcoin message and is
// used to examine the inventory being advertised by the remote peer and react
// accordingly.  We pass the message down to blockmanager which will call
// QueueMessage with any appropriate responses.
func (sp *serverPeer) OnInv(_ *peer.Peer, msg *wire.MsgInv) {
    if !cfg.BlocksOnly {
        if len(msg.InvList) > 0 {
            sp.server.syncManager.QueueInv(msg, sp.Peer)
        }
        return
    }

    newInv := wire.NewMsgInvSizeHint(uint(len(msg.InvList)))
    for _, invVect := range msg.InvList {
        if invVect.Type == wire.InvTypeTx {
            peerLog.Tracef("Ignoring tx %v in inv from %v -- "+
                "blocksonly enabled", invVect.Hash, sp)
            if sp.ProtocolVersion() >= wire.BIP0037Version {
                peerLog.Infof("Peer %v is announcing "+
                    "transactions -- disconnecting", sp)
                sp.Disconnect()
                return
            }
            continue
        }
        err := newInv.AddInvVect(invVect)
        if err != nil {
            peerLog.Errorf("Failed to add inventory vector: %v", err)
            break
        }
    }

    if len(newInv.InvList) > 0 {
        sp.server.syncManager.QueueInv(newInv, sp.Peer)
    }
}
```

> 上面的syncManager.QueueInv最后会通过blockHandler调用下面的处理方法。


#### 1.4.3.3. handleInvMsg

```go
// handleInvMsg handles inv messages from all peers.
// We examine the inventory advertised by the remote peer and act accordingly.
func (sm *SyncManager) handleInvMsg(imsg *invMsg) {
    peer := imsg.peer
    state, exists := sm.peerStates[peer]
    if !exists {
        log.Warnf("Received inv message from unknown peer %s", peer)
        return
    }

    // Attempt to find the final block in the inventory list.  There may
    // not be one.
    lastBlock := -1
    invVects := imsg.inv.InvList
    for i := len(invVects) - 1; i >= 0; i-- {
        if invVects[i].Type == wire.InvTypeBlock {
            lastBlock = i
            break
        }
    }

    // If this inv contains a block announcement, and this isn't coming from
    // our current sync peer or we're current, then update the last
    // announced block for this peer. We'll use this information later to
    // update the heights of peers based on blocks we've accepted that they
    // previously announced.
    if lastBlock != -1 && (peer != sm.syncPeer || sm.current()) {
        peer.UpdateLastAnnouncedBlock(&invVects[lastBlock].Hash)
    }

    // Ignore invs from peers that aren't the sync if we are not current.
    // Helps prevent fetching a mass of orphans.
    if peer != sm.syncPeer && !sm.current() {
        return
    }

    // If our chain is current and a peer announces a block we already
    // know of, then update their current block height.
    if lastBlock != -1 && sm.current() {
        blkHeight, err := sm.chain.BlockHeightByHash(&invVects[lastBlock].Hash)
        if err == nil {
            peer.UpdateLastBlockHeight(blkHeight)
        }
    }

    // Request the advertised inventory if we don't already have it.  Also,
    // request parent blocks of orphans if we receive one we already have.
    // Finally, attempt to detect potential stalls due to long side chains
    // we already have and request more blocks to prevent them.
    for i, iv := range invVects {
        // Ignore unsupported inventory types.
        switch iv.Type {
        case wire.InvTypeBlock:
        case wire.InvTypeTx:
        case wire.InvTypeWitnessBlock:
        case wire.InvTypeWitnessTx:
        default:
            continue
        }

        // Add the inventory to the cache of known inventory
        // for the peer.
        peer.AddKnownInventory(iv)

        // Ignore inventory when we're in headers-first mode.
        if sm.headersFirstMode {
            continue
        }

        // Request the inventory if we don't already have it.
        haveInv, err := sm.haveInventory(iv)
        if err != nil {
            log.Warnf("Unexpected failure when checking for "+
                "existing inventory during inv message "+
                "processing: %v", err)
            continue
        }
        if !haveInv {
            if iv.Type == wire.InvTypeTx {
                // Skip the transaction if it has already been
                // rejected.
                if _, exists := sm.rejectedTxns[iv.Hash]; exists {
                    continue
                }
            }

            // Ignore invs block invs from non-witness enabled
            // peers, as after segwit activation we only want to
            // download from peers that can provide us full witness
            // data for blocks.
            if !peer.IsWitnessEnabled() && iv.Type == wire.InvTypeBlock {
                continue
            }

            // Add it to the request queue.
            state.requestQueue = append(state.requestQueue, iv)
            continue
        }

        if iv.Type == wire.InvTypeBlock {
            // The block is an orphan block that we already have.
            // When the existing orphan was processed, it requested
            // the missing parent blocks.  When this scenario
            // happens, it means there were more blocks missing
            // than are allowed into a single inventory message.  As
            // a result, once this peer requested the final
            // advertised block, the remote peer noticed and is now
            // resending the orphan block as an available block
            // to signal there are more missing blocks that need to
            // be requested.
            if sm.chain.IsKnownOrphan(&iv.Hash) {
                // Request blocks starting at the latest known
                // up to the root of the orphan that just came
                // in.
                orphanRoot := sm.chain.GetOrphanRoot(&iv.Hash)
                locator, err := sm.chain.LatestBlockLocator()
                if err != nil {
                    log.Errorf("PEER: Failed to get block "+
                        "locator for the latest block: "+
                        "%v", err)
                    continue
                }
                peer.PushGetBlocksMsg(locator, orphanRoot)
                continue
            }

            // We already have the final block advertised by this
            // inventory message, so force a request for more.  This
            // should only happen if we're on a really long side
            // chain.
            if i == lastBlock {
                // Request blocks after this one up to the
                // final one the remote peer knows about (zero
                // stop hash).
                locator := sm.chain.BlockLocatorFromHash(&iv.Hash)
                peer.PushGetBlocksMsg(locator, &zeroHash)
            }
        }
    }

    // Request as much as possible at once.  Anything that won't fit into
    // the request will be requested on the next inv message.
    numRequested := 0
    gdmsg := wire.NewMsgGetData()
    requestQueue := state.requestQueue
    for len(requestQueue) != 0 {
        iv := requestQueue[0]
        requestQueue[0] = nil
        requestQueue = requestQueue[1:]

        switch iv.Type {
        case wire.InvTypeWitnessBlock:
            fallthrough
        case wire.InvTypeBlock:
            // Request the block if there is not already a pending
            // request.
            if _, exists := sm.requestedBlocks[iv.Hash]; !exists {
                sm.requestedBlocks[iv.Hash] = struct{}{}
                sm.limitMap(sm.requestedBlocks, maxRequestedBlocks)
                state.requestedBlocks[iv.Hash] = struct{}{}

                if peer.IsWitnessEnabled() {
                    iv.Type = wire.InvTypeWitnessBlock
                }

                gdmsg.AddInvVect(iv)
                numRequested++
            }

        case wire.InvTypeWitnessTx:
            fallthrough
        case wire.InvTypeTx:
            // Request the transaction if there is not already a
            // pending request.
            if _, exists := sm.requestedTxns[iv.Hash]; !exists {
                sm.requestedTxns[iv.Hash] = struct{}{}
                sm.limitMap(sm.requestedTxns, maxRequestedTxns)
                state.requestedTxns[iv.Hash] = struct{}{}

                // If the peer is capable, request the txn
                // including all witness data.
                if peer.IsWitnessEnabled() {
                    iv.Type = wire.InvTypeWitnessTx
                }

                gdmsg.AddInvVect(iv)
                numRequested++
            }
        }

        if numRequested >= wire.MaxInvPerMsg {
            break
        }
    }
    state.requestQueue = requestQueue
    if len(gdmsg.InvList) > 0 {
        peer.QueueMessage(gdmsg, nil)
    }
}
```

当syncmanager开始同步时，会设置sm.syncPeer = bestPeer。最前面，会判断它是否为同步节点发的消息，如果不是同步节点消息，并且没有同步完成，直接返回。

在for invVects循环内，会先添加当前库存信息到节点，如果在headersFirstMode情况下，说明当前节点正在同步过程中，会忽略后面的逻辑。然后检查自己节点是否有这个库存，如果haveInv=false,会做rejectedTxns和IsWitnessEnabled检查之后添加iv到requestQueue并且continue。如果已经有库存，并且是InvTypeBlock类型。会判断是否为已知的orphan block。如果是orphan block，说明对方节点有当前节点急需的orphan block，会马上生成block请求对方发送需要的block。如果刚好lastBlock==i，会请求iv.Hash之后的区块。

>**orphan block:**
如果当前节点收到一个block，但是它的前一个block在当前的chain中找不到，这个block就会添加到orphans(blockchain.orphans[hash]=block);比如当前chain为... 8, 9 , 10 .如果此时收到一个高为12的区块，它的父亲block找不到。如果调用GetOrphanRoot，会得到11的hash(&orphan.block.MsgBlock().Header.PrevBlock)。

结束for invVects之后，开始处理requestQueue。如果是block.检查是否已经在requestedBlocks中，这个map在请求区块时会标记。如果是InvTypeTx，检查requestedTxns。如果通过就添加到gdmsg中。如果一次发送的数据量超过wire.MaxInvPerMsg，就会break。requestQueue保存到state.requestQueue。在后面会接着处理。

因为网络中一直会有数据在传播，因此这个方法会间隔被调用，因此，如果此次requestQueue没有处理完成。下次还会接着处理。


