# 1. 系统启动过程
<!-- TOC -->

- [1. 系统启动过程](#1-系统启动过程)
    - [1.1. 生成配置](#11-生成配置)
        - [1.1.1. loadConfig](#111-loadconfig)
    - [1.2. 加载DB](#12-加载db)
        - [1.2.1. loadBlockDB](#121-loadblockdb)
    - [1.3. NewServer](#13-newserver)
        - [1.3.1. 创建addressManager](#131-创建addressmanager)
        - [1.3.2. initListeners](#132-initlisteners)
        - [1.3.3. 创建一个server](#133-创建一个server)
        - [1.3.4. 创建blockchain相关的索引](#134-创建blockchain相关的索引)
        - [1.3.5. mergeCheckpoints](#135-mergecheckpoints)
        - [1.3.6. 创建blockchain对象](#136-创建blockchain对象)
        - [1.3.7. 创建FeeEstimator](#137-创建feeestimator)
        - [1.3.8. 创建内存池](#138-创建内存池)
        - [1.3.9. 创建syncManager](#139-创建syncmanager)
        - [1.3.10. 创建cpu挖矿实例](#1310-创建cpu挖矿实例)
        - [1.3.11. 创建newAddressFunc](#1311-创建newaddressfunc)
        - [1.3.12. 创建connmgr](#1312-创建connmgr)
        - [1.3.13. 连结配置的固定节点](#1313-连结配置的固定节点)
        - [1.3.14. 启用RPC服务](#1314-启用rpc服务)
    - [1.4. server.start()](#14-serverstart)
        - [1.4.1. peerHandler](#141-peerhandler)

<!-- /TOC -->
## 1.1. 生成配置

### 1.1.1. loadConfig

1. 创建config对象
2. 检查并生成主目录
3. btcd.conf 生成
    1. 从相对目录下的sample-btcd.conf复制过去
4. 主网data目录生成
5. log目录生成
6. Validate database type
7. Validate any given whitelisted IP addresses and networks.
8. 配置默认的Listeners ，port :8333
9. PRC Listeners 8334
    1.  Add the default listener（default port: 8334）
    2.  生成配置的RPCUser和RPCPwd
    3.  检查RPCUser和RPCPwd，防止与限制的用户密码相同
10. Validate MinRelayTxFee(0.00001)
11. 检查配置的BlockMaxSize是否在1000到999000之间
12. 检查配置的BlockMaxWeight是否在4000到3996000之间
13. Limit the max orphan count to a sane vlue
14. Check mining addresses are valid and saved parsed versions
15. 配置cfg.dial 和cfg.lookup 方法
16. 代理配置

## 1.2. 加载DB

### 1.2.1. loadBlockDB

1. 得到dbpath
dbPath := blockDbPath(cfg.DbType) //  cfg.DbType == ffldb
2. 打开db
db, err := database.Open(cfg.DbType, dbPath, activeNetParams.Net)

```
func Open(dbType string, args ...interface{}) (DB, error) {
    drv, exists := drivers[dbType]
    if !exists {
        str := fmt.Sprintf("driver %q is not registered", dbType)
        return nil, makeError(ErrDbUnknownType, str, nil)
    }

    return drv.Open(args...)
}
```
dbPath 就是 /Btcd/data//mainnet/blocks_ffldb

drv.Open进入了database/ffldb/db.go中

如果打开失败，说明是第一次运行，就会调用接口创建一个文件：
db, err = database.Create(cfg.DbType, dbPath, activeNetParams.Net)

## 1.3. NewServer

[server.go]()

### 1.3.1. 创建addressManager

```
amgr := addrmgr.New(cfg.DataDir, btcdLookup)

addrmgr会读取默认的节点，配置文件在peers.json：
filepath.Join(dataDir, "peers.json")
```

### 1.3.2. initListeners

启用用监听tcp4和tcp6
```
const defaultServices = wire.SFNodeNetwork | wire.SFNodeBloom |
        wire.SFNodeWitness | wire.SFNodeCF

services := defaultServices
if !cfg.DisableListen {
    var err error
    listeners, nat, err = initListeners(amgr, listenAddrs, services)
    if err != nil {
        return nil, err
    }
    if len(listeners) == 0 {
        return nil, errors.New("no valid listen address")
    }
}
```
### 1.3.3. 创建一个server

```
server provides a bitcoin server for handling communications to and from
bitcoin peers.

s := server{
    chainParams:          chainParams,
    addrManager:          amgr,
    newPeers:             make(chan *serverPeer, cfg.MaxPeers),
    donePeers:            make(chan *serverPeer, cfg.MaxPeers),
    banPeers:             make(chan *serverPeer, cfg.MaxPeers),
    query:                make(chan interface{}),
    relayInv:             make(chan relayMsg, cfg.MaxPeers),
    broadcast:            make(chan broadcastMsg, cfg.MaxPeers),
    quit:                 make(chan struct{}),
    modifyRebroadcastInv: make(chan interface{}),
    peerHeightsUpdate:    make(chan updatePeerHeightsMsg),
    nat:                  nat,
    db:                   db,
    timeSource:           blockchain.NewMedianTime(),
    services:             services,
    sigCache:             txscript.NewSigCache(cfg.SigCacheMaxSize),
    hashCache:            txscript.NewHashCache(cfg.SigCacheMaxSize),
    cfCheckptCaches:      make(map[wire.FilterType][]cfHeaderKV),
}
```

### 1.3.4. 创建blockchain相关的索引

索引相关的包在btcd/blockchain/indexers/中。索引数据是通过ffldb保存到leveldb中。索引的创建都与配置有关，默认是不启用索引。

1).创建s.TxIndex和s.AddrIndex
```
s.txIndex = indexers.NewTxIndex(db)

AddrIndex依赖TxIndex
s.addrIndex = indexers.NewAddrIndex(db, chainParams)

// AddrIndex implements a transaction by address index.  That is to say, it
// supports querying all transactions that reference a given address because
// they are either crediting or debiting the address.  The returned transactions
// are ordered according to their order of appearance in the blockchain.  In
// other words, first by block height and then by offset inside the block.
//
// In addition, support is provided for a memory-only index of unconfirmed
// transactions such as those which are kept in the memory pool before inclusion
// in a block.
```

2)创建s.cfIndex
```
s.cfIndex = indexers.NewCfIndex(db, chainParams)
```
3)创建索引管理器
```
indexes为前面创建的全部索引的集合

indexManager = indexers.NewManager(db, indexes)
```
### 1.3.5. mergeCheckpoints

### 1.3.6. 创建blockchain对象
```
s.chain, err = blockchain.New(&blockchain.Config{
        DB:           s.db,
        Interrupt:    interrupt,
        ChainParams:  s.chainParams,
        Checkpoints:  checkpoints,
        TimeSource:   s.timeSource,
        SigCache:     s.sigCache,
        IndexManager: indexManager,
        HashCache:    s.hashCache,
    })

```

### 1.3.7. 创建FeeEstimator
FeeEstimator 用于评估当前交易费用。
```
// Search for a FeeEstimator state in the database. If none can be found
// or if it cannot be loaded, create a new one.
db.Update(func(tx database.Tx) error {
    metadata := tx.Metadata()
    feeEstimationData := metadata.Get(mempool.EstimateFeeDatabaseKey)
    if feeEstimationData != nil {
        // delete it from the database so that we don't try to restore the
        // same thing again somehow.
        metadata.Delete(mempool.EstimateFeeDatabaseKey)

        // If there is an error, log it and make a new fee estimator.
        var err error
        s.feeEstimator, err = mempool.RestoreFeeEstimator(feeEstimationData)

        if err != nil {
            peerLog.Errorf("Failed to restore fee estimator %v", err)
        }
    }

    return nil
})

// If no feeEstimator has been found, or if the one that has been found
// is behind somehow, create a new one and start over.
if s.feeEstimator == nil || s.feeEstimator.LastKnownHeight() != s.chain.BestSnapshot().Height {
    s.feeEstimator = mempool.NewFeeEstimator(
        mempool.DefaultEstimateFeeMaxRollback,
        mempool.DefaultEstimateFeeMinRegisteredBlocks)
}
```
三方平台：
https://www.buybitcoinworldwide.com/fee-calculator/

### 1.3.8. 创建内存池

内存池用于存放未确认的交易，在挖矿时会从这里取。

mempool provides a policy-enforced pool of unmined bitcoin transactions.
A key responsbility of the bitcoin network is mining user-generated transactions into blocks. 
 In order to facilitate this, the mining process relies on having a
readily-available source of transactions to include in a block that is being solved.

```
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

### 1.3.9. 创建syncManager
syncManager用于节点间消息同步，如区块。

SyncManager is used to communicate block related messages with peers. The SyncManager is started as by executing Start() in a goroutine. Once started,it selects peers to sync from and starts the initial block download. Once the
chain is in sync, the SyncManager handles incoming block and header notifications and relays announcements of new blocks to peers.

```
s.syncManager, err = netsync.New(&netsync.Config{
    PeerNotifier:       &s,
    Chain:              s.chain,
    TxMemPool:          s.txMemPool,
    ChainParams:        s.chainParams,
    DisableCheckpoints: cfg.DisableCheckpoints,
    MaxPeers:           cfg.MaxPeers,
    FeeEstimator:       s.feeEstimator,
})
```

### 1.3.10. 创建cpu挖矿实例

```
Create the mining policy and block template generator based on the configuration options.
NOTE: The CPU miner relies on the mempool, so the mempool has to be created before calling the function to create the CPU miner.

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
```

### 1.3.11. 创建newAddressFunc
newAddressFunc用于得到一个新的节点地址，里面会用到随机算法。
```
Only setup a function to return new addresses to connect to when not running in connect-only mode.  
The simulation network is always in connect-only mode since it is only intended to connect to specified peers 
and actively avoid advertising and connecting to discovered peers in order to prevent it 
from becoming a public test network.

var newAddressFunc func() (net.Addr, error)
if !cfg.SimNet && len(cfg.ConnectPeers) == 0 {
    newAddressFunc = func() (net.Addr, error) {
        for tries := 0; tries < 100; tries++ {
            addr := s.addrManager.GetAddress()
            ...
            addrString := addrmgr.NetAddressKey(addr.NetAddress())
            return addrStringToNetAddr(addrString)
        }

        return nil, errors.New("no valid connect address")
    }
}
```
### 1.3.12. 创建connmgr

connmgr 用于创建并维护连结，里面有两个重要的参数OnConnection和OnAccept用于创建peer.
```
// Create a connection manager.
targetOutbound := defaultTargetOutbound
if cfg.MaxPeers < targetOutbound {
    targetOutbound = cfg.MaxPeers
}
cmgr, err := connmgr.New(&connmgr.Config{
    Listeners:      listeners,
    OnAccept:       s.inboundPeerConnected,
    RetryDuration:  connectionRetryInterval,
    TargetOutbound: uint32(targetOutbound),
    Dial:           btcdDial,
    OnConnection:   s.outboundPeerConnected,
    GetNewAddress:  newAddressFunc,
})
if err != nil {
    return nil, err
}
s.connManager = cmgr
```

### 1.3.13. 连结配置的固定节点

```
// Start up persistent peers.
permanentPeers := cfg.ConnectPeers
if len(permanentPeers) == 0 {
    permanentPeers = cfg.AddPeers
}
for _, addr := range permanentPeers {
    netAddr, err := addrStringToNetAddr(addr)
    if err != nil {
        return nil, err
    }

    go s.connManager.Connect(&connmgr.ConnReq{
        Addr:      netAddr,
        Permanent: true,
    })
}
```

### 1.3.14. 启用RPC服务

```
if !cfg.DisableRPC {
    // Setup listeners for the configured RPC listen addresses and
    // TLS settings.
    rpcListeners, err := setupRPCListeners()
    if err != nil {
        return nil, err
    }
    if len(rpcListeners) == 0 {
        return nil, errors.New("RPCS: No valid listen address")
    }

    s.rpcServer, err = newRPCServer(&rpcserverConfig{
        Listeners:    rpcListeners,
        StartupTime:  s.startupTime,
        ConnMgr:      &rpcConnManager{&s},
        SyncMgr:      &rpcSyncMgr{&s, s.syncManager},
        TimeSource:   s.timeSource,
        Chain:        s.chain,
        ChainParams:  chainParams,
        DB:           db,
        TxMemPool:    s.txMemPool,
        Generator:    blockTemplateGenerator,
        CPUMiner:     s.cpuMiner,
        TxIndex:      s.txIndex,
        AddrIndex:    s.addrIndex,
        CfIndex:      s.cfIndex,
        FeeEstimator: s.feeEstimator,
    })
    if err != nil {
        return nil, err
    }

    // Signal process shutdown when the RPC server requests it.
    go func() {
        <-s.rpcServer.RequestedProcessShutdown()
        shutdownRequestChannel <- struct{}{}
    }()
}
```

## 1.4. server.start()

start会根据配置启用几个独立的模块。

- go s.peerHandler()
- go s.upnpUpdateThread()
- go s.rebroadcastHandler()
- s.rpcServer.Start()
- s.cpuMiner.Start()

**code:**
```
// Start begins accepting connections from peers.
func (s *server) Start() {
    // Already started?
    if atomic.AddInt32(&s.started, 1) != 1 {
        return
    }

    srvrLog.Trace("Starting server")

    // Server startup time. Used for the uptime command for uptime calculation.
    s.startupTime = time.Now().Unix()

    // Start the peer handler which in turn starts the address and block
    // managers.
    s.wg.Add(1)
    go s.peerHandler()

    if s.nat != nil {
        s.wg.Add(1)
        go s.upnpUpdateThread()
    }

    if !cfg.DisableRPC {
        s.wg.Add(1)

        // Start the rebroadcastHandler, which ensures user tx received by
        // the RPC server are rebroadcast until being included in a block.
        go s.rebroadcastHandler()

        s.rpcServer.Start()
    }

    // Start the CPU miner if generation is enabled.
    if cfg.Generate {
        s.cpuMiner.Start()
    }
}
```

### 1.4.1. peerHandler

peerHandler启动了节点相关的三个管理服务。

- s.addrManager.Start()
- s.syncManager.Start()
- go s.connManager.Start()

同时，它会一直循环去读取管道中的节点消息。调用相关的处理器处理。
直到收到退出命令，断开所有节点的连结。最后，调用s.wg.Done()终止等待。

**code:**
```
// peerHandler is used to handle peer operations such as adding and removing
// peers to and from the server, banning peers, and broadcasting messages to
// peers.  It must be run in a goroutine.

func (s *server) peerHandler() {
    // Start the address manager and sync manager, both of which are needed
    // by peers.  This is done here since their lifecycle is closely tied
    // to this handler and rather than adding more channels to sychronize
    // things, it's easier and slightly faster to simply start and stop them
    // in this handler.
    s.addrManager.Start()
    s.syncManager.Start()

    srvrLog.Tracef("Starting peer handler")

    state := &peerState{
        inboundPeers:    make(map[int32]*serverPeer),
        persistentPeers: make(map[int32]*serverPeer),
        outboundPeers:   make(map[int32]*serverPeer),
        banned:          make(map[string]time.Time),
        outboundGroups:  make(map[string]int),
    }

    if !cfg.DisableDNSSeed {
        // Add peers discovered through DNS to the address manager.
        connmgr.SeedFromDNS(activeNetParams.Params, defaultRequiredServices,
            btcdLookup, func(addrs []*wire.NetAddress) {
                // Bitcoind uses a lookup of the dns seeder here. This
                // is rather strange since the values looked up by the
                // DNS seed lookups will vary quite a lot.
                // to replicate this behaviour we put all addresses as
                // having come from the first one.
                s.addrManager.AddAddresses(addrs, addrs[0])
            })
    }
    go s.connManager.Start()

out:
    for {
        select {
        // New peers connected to the server.
        case p := <-s.newPeers:
            s.handleAddPeerMsg(state, p)

        // Disconnected peers.
        case p := <-s.donePeers:
            s.handleDonePeerMsg(state, p)

        // Block accepted in mainchain or orphan, update peer height.
        case umsg := <-s.peerHeightsUpdate:
            s.handleUpdatePeerHeights(state, umsg)

        // Peer to ban.
        case p := <-s.banPeers:
            s.handleBanPeerMsg(state, p)

        // New inventory to potentially be relayed to other peers.
        case invMsg := <-s.relayInv:
            s.handleRelayInvMsg(state, invMsg)

        // Message to broadcast to all connected peers except those
        // which are excluded by the message.
        case bmsg := <-s.broadcast:
            s.handleBroadcastMsg(state, &bmsg)

        case qmsg := <-s.query:
            s.handleQuery(state, qmsg)

        case <-s.quit:
            // Disconnect all peers on server shutdown.
            state.forAllPeers(func(sp *serverPeer) {
                srvrLog.Tracef("Shutdown peer %s", sp)
                sp.Disconnect()
            })
            break out
        }
    }

    s.connManager.Stop()
    s.syncManager.Stop()
    s.addrManager.Stop()

    // Drain channels before exiting so nothing is left waiting around
    // to send.
cleanup:
    for {
        select {
        case <-s.newPeers:
        case <-s.donePeers:
        case <-s.peerHeightsUpdate:
        case <-s.relayInv:
        case <-s.broadcast:
        case <-s.query:
        default:
            break cleanup
        }
    }
    s.wg.Done()
    srvrLog.Tracef("Peer handler done")
    }
```


