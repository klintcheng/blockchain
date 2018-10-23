
# 1. bitcoin(btcd)技术原理
<!-- TOC -->

- [1. bitcoin(btcd)技术原理](#1-bitcoinbtcd技术原理)
    - [1.1. 默认启动过程](#11-默认启动过程)
    - [1.2. 包结构](#12-包结构)
        - [1.2.1. /addrmgr](#121-addrmgr)
        - [1.2.2. /blockchain](#122-blockchain)
        - [1.2.3. /btcec](#123-btcec)
        - [1.2.4. /btcjson](#124-btcjson)
        - [1.2.5. /chaincfg](#125-chaincfg)
        - [1.2.6. /cmd](#126-cmd)
        - [1.2.7. /connmgr](#127-connmgr)
        - [1.2.8. /database](#128-database)
        - [1.2.9. /limits](#129-limits)
        - [1.2.10. /mempool](#1210-mempool)
        - [1.2.11. /mining](#1211-mining)
        - [1.2.12. /netsync](#1212-netsync)
        - [1.2.13. /peer](#1213-peer)
        - [1.2.14. /rpcclient](#1214-rpcclient)
        - [1.2.15. /txscript](#1215-txscript)
        - [1.2.16. /wire](#1216-wire)
    - [1.3. 主要代码](#13-主要代码)
        - [1.3.1. btcd.go](#131-btcdgo)
        - [1.3.2. config.go](#132-configgo)
        - [1.3.3. server.go](#133-servergo)
            - [serverPeer](#serverpeer)
            - [serverPeer](#serverpeer-1)

<!-- /TOC -->
## 1.1. 默认启动过程

```
<!-- 启动，加载配置 -->
API server listening at: 127.0.0.1:21450
2018-10-23 10:56:04.642 [INF] BTCD: Version 0.12.0-beta
2018-10-23 10:56:04.642 [INF] BTCD: Loading block database from '/Users/klint/Library/Application Support/Btcd/data/mainnet/blocks_ffldb'
<!-- 加载DB, leveldb -->
2018-10-23 10:56:04.649 [INF] BTCD: Block database loaded
2018-10-23 10:56:04.667 [INF] INDX: Committed filter index is enabled
2018-10-23 10:56:04.668 [INF] INDX: Catching up indexes from height -1 to 0
2018-10-23 10:56:04.668 [INF] INDX: Indexes caught up to height 0
<!-- 当前节点区块链信息 -->
2018-10-23 10:56:04.668 [INF] CHAN: Chain state (height 0, hash 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f, totaltx 1, work 4295032833)
2018-10-23 10:56:04.669 [INF] RPCS: Generating TLS certificates...
2018-10-23 10:56:04.734 [INF] RPCS: Done generating TLS certificates
<!-- 启用RPC监听 -->
2018-10-23 10:56:04.773 [INF] RPCS: RPC server listening on 127.0.0.1:8334
<!-- 尝试读取节点 -->
2018-10-23 10:56:04.773 [INF] AMGR: Loaded 0 addresses from file '/Users/klint/Library/Application Support/Btcd/data/mainnet/peers.json'
2018-10-23 10:56:04.773 [INF] RPCS: RPC server listening on [::1]:8334
<!-- 得到种子节点 -->
2018-10-23 10:56:04.777 [INF] CMGR: 40 addresses found from DNS seed seed.bitcoin.sipa.be
2018-10-23 10:56:04.778 [INF] CMGR: Server listening on [::]:8333
2018-10-23 10:56:04.779 [INF] CMGR: 33 addresses found from DNS seed dnsseed.bluematt.me
2018-10-23 10:56:04.780 [INF] CMGR: Server listening on 0.0.0.0:8333
2018-10-23 10:56:04.782 [INF] CMGR: 38 addresses found from DNS seed dnsseed.bitcoin.dashjr.org
2018-10-23 10:56:04.783 [INF] CMGR: 38 addresses found from DNS seed seed.bitcoin.jonasschnelli.ch
2018-10-23 10:56:04.789 [INF] CMGR: 40 addresses found from DNS seed seed.bitnodes.io
2018-10-23 10:56:04.789 [INF] CMGR: 50 addresses found from DNS seed seed.bitcoinstats.com
<!-- 建立节点连结 -->
2018-10-23 10:56:05.269 [INF] SYNC: New valid peer 206.189.225.250:8333 (outbound) (/Satoshi:0.16.3/)
<!-- 马上开始同步节点区块 -->
2018-10-23 10:56:05.270 [INF] SYNC: Syncing to block height 546957 from peer 206.189.225.250:8333
<!-- 下载区块头 -->
2018-10-23 10:56:05.270 [INF] SYNC: Downloading headers for blocks 1 to 11111 from peer 206.189.225.250:8333
2018-10-23 10:56:05.299 [INF] SYNC: New valid peer 144.76.91.109:8333 (outbound) (/Satoshi:0.17.0/)
2018-10-23 10:56:05.490 [INF] SYNC: New valid peer 47.90.244.228:8333 (outbound) (/Satoshi:0.16.3/)
2018-10-23 10:56:05.627 [INF] SYNC: New valid peer 135.23.196.24:8333 (outbound) (/Satoshi:0.16.2/)
2018-10-23 10:56:08.790 [INF] SYNC: New valid peer 149.202.84.91:8333 (outbound) (/Satoshi:0.16.3/)
2018-10-23 10:56:10.786 [INF] SYNC: New valid peer 186.136.146.129:8333 (outbound) (/Satoshi:0.17.0/)
2018-10-23 10:56:36.788 [INF] SYNC: New valid peer 181.166.168.210:8333 (outbound) (/Satoshi:0.17.0/)
2018-10-23 10:56:36.852 [INF] SYNC: New valid peer 159.89.195.201:8333 (outbound) (/Satoshi:0.15.1/)
2018-10-23 10:57:30.873 [INF] SYNC: Verified downloaded block header against checkpoint at height 11111/hash 0000000069e244f73d78e8fd29ba2fd2ed618bd6fa2ee92559f542fdb26e7c1d
<!-- 开始加载区块 -->
2018-10-23 10:57:30.873 [INF] SYNC: Received 11111 block headers: Fetching blocks
2018-10-23 10:57:40.953 [INF] SYNC: Processed 539 blocks in the last 10.07s (548 transactions, height 539, 2009-01-15 12:46:34 +0800 CST)
2018-10-23 10:57:51.040 [INF] SYNC: Processed 1205 blocks in the last 10.08s (1223 transactions, height 1744, 2009-01-25 16:15:05 +0800 CST)
2018-10-23 10:58:01.280 [INF] SYNC: Processed 1028 blocks in the last 10.23s (1035 transactions, height 2772, 2009-02-03 05:47:37 +0800 CST)
2018-10-23 10:58:11.428 [INF] SYNC: Processed 515 blocks in the last 10.14s (531 transactions, height 3287, 2009-02-07 07:27:37 +0800 CST)
2018-10-23 10:58:21.464 [INF] SYNC: Processed 464 blocks in the last 10.03s (470 transactions, height 3751, 2009-02-10 21:21:35 +0800 CST)
2018-10-23 10:58:31.694 [INF] SYNC: Processed 1567 blocks in the last 10.22s (1578 transactions, height 5318, 2009-02-23 16:50:59 +0800 CST)
2018-10-23 10:58:41.697 [INF] SYNC: Processed 2100 blocks in the last 10s (2112 transactions, height 7418, 2009-03-14 21:15:22 +0800 CST)
2018-10-23 10:58:51.777 [INF] SYNC: Processed 1714 blocks in the last 10.07s (1724 transactions, height 9132, 2009-03-30 03:16:14 +0800 CST)
2018-10-23 10:59:01.778 [INF] SYNC: Processed 1751 blocks in the last 10s (1758 transactions, height 10883, 2009-04-14 13:19:35 +0800 CST)
<!-- 检查checkpoint，块高为11111 -->
2018-10-23 10:59:02.717 [INF] CHAN: Verified checkpoint at height 11111/block 0000000069e244f73d78e8fd29ba2fd2ed618bd6fa2ee92559f542fdb26e7c1d
<!-- 开始下载区块链其它的数据 -->
2018-10-23 10:59:02.718 [INF] SYNC: Downloading headers for blocks 11112 to 33333 from peer 206.189.225.250:8333

```

## 1.2. 包结构

### 1.2.1. /addrmgr
地址管理，维护可用节点地址信息。
```
In order maintain the peer-to-peer Bitcoin network, there needs to be a source
of addresses to connect to as nodes come and go.  The Bitcoin protocol provides
the getaddr and addr messages to allow peers to communicate known addresses with
each other.  However, there needs to a mechanism to store those results and
select peers from them.  It is also important to note that remote peers can't
be trusted to send valid peers nor attempt to provide you with only peers they
control with malicious intent.
```
[detail](https://github.com/btcsuite/btcd/blob/master/addrmgr/doc.go)

### 1.2.2. /blockchain

实现区块处理及chain选择规则
```
Package blockchain implements bitcoin block handling and chain selection rules.
```
[detail](https://github.com/btcsuite/btcd/tree/master/blockchain)
### 1.2.3. /btcec
非对称加密算法：椭圆曲线加密算法
```
Package btcec implements elliptic curve cryptography needed for working with Bitcoin (secp256k1 only for now). It is designed so that it may be used with the standard crypto/ecdsa packages provided with go. 
```

[detail](https://github.com/btcsuite/btcd/tree/master/btcec)

### 1.2.4. /btcjson

```
Package btcjson implements concrete types for marshalling to and from the bitcoin JSON-RPC API. A comprehensive suite of tests is provided to ensure proper functionality.
```

[detail](https://github.com/btcsuite/btcd/tree/master/btcjson)

### 1.2.5. /chaincfg
```
Package chaincfg defines chain configuration parameters for the three standard Bitcoin networks and provides the ability for callers to define their own custom Bitcoin networks.
```

[detail](https://github.com/btcsuite/btcd/tree/master/chaincfg)

### 1.2.6. /cmd
命令包，主要实现了btcctl，通过命令调用节点接口。
```
Usage:
  btcctl [OPTIONS] <command> <args...>

btcctl -h 显示可用选项
btcctl -l 显示命令
如：
getaccount "address"
getaccountaddress "account"
getaddressesbyaccount "account"
getbalance ("account" minconf=1)
getnewaddress ("account")
```
[detail](https://github.com/btcsuite/btcd/tree/master/cmd)

### 1.2.7. /connmgr
连接创建器
```
Connection Manager handles all the general connection concerns such as maintaining 
a set number of outbound connections, sourcing peers, banning, limiting max connections, tor lookup, etc.

The package provides a generic connection manager which is able to accept connection requests 
from a source or a set of given addresses, dial them and notify the caller on connections. 
The main intended use is to initialize a pool of active connections and maintain them 
to remain connected to the P2P network.
```

In addition the connection manager provides the following utilities:

- Notifications on connections or disconnections
- Handle failures and retry new addresses from the source
- Connect only to specified addresses
- Permanent connections with increasing backoff retry timers
- Disconnect or Remove an established connection

[detail](https://github.com/btcsuite/btcd/tree/master/connmgr)

### 1.2.8. /database
数据库包,提供db接口，实现了ffldb。它用leveldb实现metadata保存，用文件保存区块信息。
```
The default backend, ffldb, has a strong focus on speed, efficiency, and robustness. 
It makes use of leveldb for the metadata, flat files for block storage, 
and strict checksums in key areas to ensure data integrity.
```
**Feature Overview**
- Key/value metadata store
- Bitcoin block storage
- Efficient retrieval of block headers and regions (transactions, scripts, etc)
- Read-only and read-write transactions with both manual and managed modes
- Nested buckets
- Iteration support including cursors with seek capability
- Supports registration of backend databases
- Comprehensive test coverage

[detail](https://github.com/btcsuite/btcd/tree/master/database)

### 1.2.9. /limits
实现SetLimits()。在启用main中会调用

[detail](https://github.com/btcsuite/btcd/tree/master/limits)

### 1.2.10. /mempool
交易内存池
```
Package mempool provides a policy-enforced pool of unmined bitcoin transactions.

A key responsbility of the bitcoin network is mining user-generated transactions into blocks.
 In order to facilitate this, the mining process relies on having a readily-available source of 
 transactions to include in a block that is being solved.
```
**Feature Overview**
- Maintain a pool of fully validated transactions
- Orphan transaction support (transactions that spend from unknown outputs)
- Configurable transaction acceptance policy
- Additional metadata tracking for each transaction
- Manual control of transaction removal

[detail](https://github.com/btcsuite/btcd/tree/master/mempool)

### 1.2.11. /mining
cpu挖矿实现
[detail](https://github.com/btcsuite/btcd/tree/master/mining)

### 1.2.12. /netsync
```
This package implements a concurrency safe block syncing protocol. 
The SyncManager communicates with connected peers to perform an initial block download, 
keep the chain and unconfirmed transaction pool in sync, and announce new blocks connected to the chain. 
Currently the sync manager selects a single sync peer that it downloads all blocks from 
until it is up to date with the longest chain the sync peer is aware of.
```
[detail](https://github.com/btcsuite/btcd/tree/master/netsync)

### 1.2.13. /peer
```
This package builds upon the wire package, which provides the fundamental primitives necessary to speak the bitcoin wire protocol, 
in order to simplify the process of creating fully functional peers. 
In essence, it provides a common base for creating concurrent safe fully validating nodes, 
Simplified Payment Verification (SPV) nodes, proxies, etc.
```
**Feature Overview**

 - Provides a basic concurrent safe bitcoin peer for handling bitcoin
   communications via the peer-to-peer protocol
 - Full duplex reading and writing of bitcoin protocol messages
 - Automatic handling of the initial handshake process including protocol version negotiation
 - Asynchronous message queueing of outbound messages with optional channel for notification when the message is actually sent
 - Flexible peer configuration
   - Caller is responsible for creating outgoing connections and listening for incoming connections so they have flexibility to establish connections as they see fit (proxies, etc)
   - User agent name and version
   - Bitcoin network
   - Service support signalling (full nodes, bloom filters, etc)
   - Maximum supported protocol version
   - Ability to register callbacks for handling bitcoin protocol messages
 - Inventory message batching and send trickling with known inventory detection
   and avoidance
 - Automatic periodic keep-alive pinging and pong responses
 - Random nonce generation and self connection detection
 - Proper handling of bloom filter related commands when the caller does not specify the related flag to signal support
   - Disconnects the peer when the protocol version is high enough
   - Does not invoke the related callbacks for older protocol versions
 - Snapshottable peer statistics such as the total number of bytes read and written, the remote address, user agent, and negotiated protocol version
 - Helper functions pushing addresses, getblocks, getheaders, and reject
   messages
   - These could all be sent manually via the standard message output function, but the helpers provide additional nice functionality such as duplicate filtering and address randomization
 - Ability to wait for shutdown/disconnect
 - Comprehensive test coverage
  
[detail](https://github.com/btcsuite/btcd/tree/master/peer)

### 1.2.14. /rpcclient
封装的RPC调用客户端
```
rpcclient implements a Websocket-enabled Bitcoin JSON-RPC client package written in Go. 
It provides a robust and easy to use client for interfacing with a Bitcoin RPC server 
that uses a btcd/bitcoin core compatible Bitcoin JSON-RPC API.
```

[detail](https://github.com/btcsuite/btcd/tree/master/rpcclient)

### 1.2.15. /txscript
提供交易中使用的基于堆的语言实现,非图灵完备。
```
Bitcoin provides a stack-based, FORTH-like language for the scripts in the bitcoin transactions. 
This language is not turing complete although it is still fairly powerful. 
A description of the language can be found at https://en.bitcoin.it/wiki/Script
```

[detail](https://github.com/btcsuite/btcd/tree/master/txscript)

### 1.2.16. /wire
数据交换网络协议
```
The bitcoin protocol consists of exchanging messages between peers. Each message is preceded by a header which identifies 
information about it such as which bitcoin network it is a part of, its type, how big it is, and a checksum to verify validity. 
All encoding and decoding of message headers is handled by this package.

To accomplish this, there is a generic interface for bitcoin messages named Message which allows messages of any type to be read,
 written, or passed around through channels, functions, etc. In addition, concrete implementations of most of the currently 
 supported bitcoin messages are provided. For these supported messages, all of the details of marshalling and unmarshalling to 
 and from the wire using bitcoin encoding are handled so the caller doesn't have to concern themselves with the specifics.

```

message.go:
- ReadMessage
- WriteMessage

[detail](https://github.com/btcsuite/btcd/tree/master/wire)

**BlockHeader**
```
// BlockHeader defines information about a block and is used in the bitcoin
// block (MsgBlock) and headers (MsgHeaders) messages.
type BlockHeader struct {
    // Version of the block.  This is not the same as the protocol version.
    Version int32

    // Hash of the previous block header in the block chain.
    PrevBlock chainhash.Hash

    // Merkle tree reference to hash of all transactions for the block.
    MerkleRoot chainhash.Hash

    // Time the block was created.  This is, unfortunately, encoded as a
    // uint32 on the wire and therefore is limited to 2106.
    Timestamp time.Time

    // Difficulty target for the block.
    Bits uint32

    // Nonce used to generate the block.
    Nonce uint32
}
```
## 1.3. 主要代码

### 1.3.1. btcd.go 
main入口，处理流程

- runtime.GOMAXPROCS(runtime.NumCPU())
- Up some limits.
```
if err := limits.SetLimits(); err != nil {
    fmt.Fprintf(os.Stderr, "failed to set limits: %v\n", err)
    os.Exit(1)
}
```
- Load configuration and parse command line.  
```
This function also initializes logging and configures it accordingly.

tcfg, _, err := loadConfig()
```
- Load the block database.
```
loadBlockDB loads (or creates when needed) the block database taking into account the selected database backend
 and returns a handle to it.  It also contains additional logic such warning the user if there are multiple databases
  which consume space on the file system and ensuring the regression test database is clean when in regression test mode.
```
- Create server and start it.
```
// Create server and start it.
server, err := newServer(cfg.Listeners, db, activeNetParams.Params,
    interrupt)
if err != nil {
    // TODO: this logging could do with some beautifying.
    btcdLog.Errorf("Unable to start server on %v: %v",
        cfg.Listeners, err)
    return err
}
defer func() {
    btcdLog.Infof("Gracefully shutting down the server...")
    server.Stop()
    server.WaitForShutdown()
    srvrLog.Infof("Server shutdown complete")
}()
server.Start()
```

### 1.3.2. config.go 
配置管理

```

loadConfig initializes and parses the config using a config file and command line options.
The configuration proceeds as follows:
1) Start with a default config with sane settings
2) Pre-parse the command line to check for an alternative config file
3) Load configuration file overwriting defaults with any specified options
4) Parse CLI options and overwrite/add any specified options

The above results in btcd functioning properly without any config settings 
while still allowing the user to override settings with config files and command line options.  
Command line options always take precedence.
```

### 1.3.3. server.go 
server是最重要的部分。它主要完成如下功能

- 初始化所有需要的组件
- 启动相关服务
- 实现组件的通信功能

主要结构体：server 和serverPeer

#### serverPeer
```
// server provides a bitcoin server for handling communications to and from
// bitcoin peers.
type server struct {
    // The following variables must only be used atomically.
    // Putting the uint64s first makes them 64-bit aligned for 32-bit systems.
    bytesReceived uint64 // Total bytes received from all peers since start.
    bytesSent     uint64 // Total bytes sent by all peers since start.
    started       int32
    shutdown      int32
    shutdownSched int32
    startupTime   int64

    chainParams          *chaincfg.Params
    addrManager          *addrmgr.AddrManager
    connManager          *connmgr.ConnManager
    sigCache             *txscript.SigCache
    hashCache            *txscript.HashCache
    rpcServer            *rpcServer
    syncManager          *netsync.SyncManager
    chain                *blockchain.BlockChain
    txMemPool            *mempool.TxPool
    cpuMiner             *cpuminer.CPUMiner
    modifyRebroadcastInv chan interface{}
    newPeers             chan *serverPeer
    donePeers            chan *serverPeer
    banPeers             chan *serverPeer
    query                chan interface{}
    relayInv             chan relayMsg
    broadcast            chan broadcastMsg
    peerHeightsUpdate    chan updatePeerHeightsMsg
    wg                   sync.WaitGroup
    quit                 chan struct{}
    nat                  NAT
    db                   database.DB
    timeSource           blockchain.MedianTimeSource
    services             wire.ServiceFlag

    // The following fields are used for optional indexes.  They will be nil
    // if the associated index is not enabled.  These fields are set during
    // initial creation of the server and never changed afterwards, so they
    // do not need to be protected for concurrent access.
    txIndex   *indexers.TxIndex
    addrIndex *indexers.AddrIndex
    cfIndex   *indexers.CfIndex

    // The fee estimator keeps track of how long transactions are left in
    // the mempool before they are mined into blocks.
    feeEstimator *mempool.FeeEstimator

    // cfCheckptCaches stores a cached slice of filter headers for cfcheckpt
    // messages for each filter type.
    cfCheckptCaches    map[wire.FilterType][]cfHeaderKV
    cfCheckptCachesMtx sync.RWMutex
}
```

#### serverPeer

实现节点通信相关的接口

```
// serverPeer extends the peer to maintain state shared by the server and
// the blockmanager.
type serverPeer struct {
    // The following variables must only be used atomically
    feeFilter int64

    *peer.Peer

    connReq        *connmgr.ConnReq
    server         *server
    persistent     bool
    continueHash   *chainhash.Hash
    relayMtx       sync.Mutex
    disableRelayTx bool
    sentAddrs      bool
    isWhitelisted  bool
    filter         *bloom.Filter
    knownAddresses map[string]struct{}
    banScore       connmgr.DynamicBanScore
    quit           chan struct{}
    // The following chans are used to sync blockmanager and server.
    txProcessed    chan struct{}
    blockProcessed chan struct{}
}
```

详见第二章
