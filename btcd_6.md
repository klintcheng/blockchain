
# 1. 节点peer
<!-- TOC -->

- [1. 节点peer](#1-节点peer)
    - [1.1. peer overview](#11-peer-overview)
    - [1.2. peer.start](#12-peerstart)
    - [1.3. peer握手](#13-peer握手)
        - [1.3.1. 发送version消息](#131-发送version消息)
        - [1.3.2. 读version消息](#132-读version消息)
    - [1.4. 消息收发](#14-消息收发)
        - [1.4.1. 消息结构](#141-消息结构)
        - [1.4.2. 发消息](#142-发消息)
            - [1.4.2.1. 写outputQueue](#1421-写outputqueue)
            - [1.4.2.2. queueHandler](#1422-queuehandler)
            - [1.4.2.3. outHandler](#1423-outhandler)
            - [1.4.2.4. 消息发送到节点](#1424-消息发送到节点)
        - [1.4.3. 收到消息](#143-收到消息)
            - [1.4.3.1. inHandler](#1431-inhandler)
            - [1.4.3.2. 消息处理](#1432-消息处理)
            - [1.4.3.3. 消息回复](#1433-消息回复)
    - [1.5. 心跳机制](#15-心跳机制)
        - [1.5.1. pingHandler](#151-pinghandler)
        - [1.5.2. 处理心跳](#152-处理心跳)
    - [1.6. 消息间隔处理](#16-消息间隔处理)
        - [1.6.1. 写stallControl](#161-写stallcontrol)
            - [1.6.1.1. 发消息时](#1611-发消息时)
            - [1.6.1.2. 收到消息时](#1612-收到消息时)
        - [1.6.2. stallHandler](#162-stallhandler)

<!-- /TOC -->

## 1.1. peer overview

The overall data flow of a peer is split into **3 goroutines**.  Inbound
messages are read via the **inHandler** goroutine and generally dispatched to
their own handler.  For inbound data-related messages such as blocks,
transactions, and inventory, the data is handled by the corresponding
message handlers.  The data flow for outbound messages is split into 2
goroutines, **queueHandler** and **outHandler**.  The first, queueHandler, is used
as a way for external entities to queue messages, by way of the QueueMessage
function, quickly regardless of whether the peer is currently sending or not.
It acts as the traffic cop between the external world and the actual
goroutine which writes to the network socket.

Peer provides a basic concurrent safe bitcoin peer for handling bitcoin
communications via the peer-to-peer protocol.  It provides **full duplex
reading and writing**, automatic handling of the **initial handshake** process,
querying of **usage statistics **and other information about the remote peer such
as its address, user agent, and protocol version, output message queuing,
inventory trickling, and the ability to dynamically register and unregister
callbacks for handling bitcoin protocol messages.

Outbound messages are typically queued via **QueueMessage** or **QueueInventory**.
QueueMessage is intended for all messages, including responses to data such
as blocks and transactions.  QueueInventory, on the other hand, is only
intended for relaying inventory as it employs a trickling mechanism to batch
the inventory together.  However, some helper functions for pushing messages
of specific types that typically require common special handling are
provided as a convenience.

## 1.2. peer.start

当一个serverpeer创建完成之后，会关联一个连接，然后调用start启用节点。
AssociateConnection在前面已经提到过。

```go
// AssociateConnection associates the given conn to the peer.   Calling this
// function when the peer is already connected will have no effect.
func (p *Peer) AssociateConnection(conn net.Conn) {
    ...
    go func() {
        if err := p.start(); err != nil {
            log.Debugf("Cannot start peer %v: %v", p, err)
            p.Disconnect()
        }
    }()
}
```

**p.start():**

```go
// start begins processing input and output messages.
func (p *Peer) start() error {
    log.Tracef("Starting peer %s", p)

    negotiateErr := make(chan error, 1)
    go func() {
        if p.inbound {
            negotiateErr <- p.negotiateInboundProtocol()
        } else {
            negotiateErr <- p.negotiateOutboundProtocol()
        }
    }()

    // Negotiate the protocol within the specified negotiateTimeout.
    select {
    case err := <-negotiateErr:
        if err != nil {
            p.Disconnect()
            return err
        }
    case <-time.After(negotiateTimeout):
        p.Disconnect()
        return errors.New("protocol negotiation timeout")
    }
    log.Debugf("Connected to %s", p.Addr())

    // The protocol has been negotiated successfully so start processing input
    // and output messages.
    go p.stallHandler()
    go p.inHandler()
    go p.queueHandler()
    go p.outHandler()
    go p.pingHandler()

    // Send our verack message now that the IO processing machinery has started.
    p.QueueMessage(wire.NewMsgVerAck(), nil)
    return nil
}
```

> peer会启用一些处理器：
> 1. stallHandler： 消息回复延时处理  
    handles stall detection for the peer.  This entails keeping track of expected responses and assigning them deadlines while accounting for the time spent in callbacks.  It must be run as a goroutine.
    
> 2. inHandler: 处理进入的消息  
    handles all incoming messages for the peer.  It must be run as a goroutine.

> 3. queueHandler： 处理节点出去的消息，写到sendQueue队列  
    handles the queuing of outgoing data for the peer. This runs as a muxer for various sources of input so we can ensure that server and peer handlers will not block on us sending a message.  That data is then passed on to outHandler to be actually written.
    
> 4. outHandler： 处理sendQueue消息，发送出去  
    handles all outgoing messages for the peer.  It must be run as a goroutine.  It uses a buffered channel to serialize output messages while allowing the sender to continue running asynchronously

> 5. pingHandler：心跳处理  
    periodically pings the peer.  It must be run as a goroutine.

## 1.3. peer握手

当一个节点连接之后，它会发送自己的版本消息给对方，对方收到消息之后，也会回复自己的信息。


**Once one or more connections are established, the new node will send an addr message containing its own IP address to its neighbors. The neighbors will, in turn, forward the addr message to their neighbors, ensuring that the newly connected node becomes well known and better connected. Additionally, the newly connected node can send getaddr to the neighbors, asking them to return a list of IP addresses of other peers. That way, a node can find peers to connect to and advertise its existence on the network for other nodes to find it. Address propagation and discovery shows the address discovery protocol. [mastering bitcoin]**


在peer.start()中，首先会启用一个goroutine去处理握手。

```go
negotiateErr := make(chan error, 1)
go func() {
    if p.inbound {
        negotiateErr <- p.negotiateInboundProtocol()
    } else {
        negotiateErr <- p.negotiateOutboundProtocol()
    }
}()
```

如果这个节点B是当前节点A接连过去的，那么它会调用negotiateOutboundProtocol。negotiateOutboundProtocol与negotiateInboundProtocol是一个必定是个相反的过程。

```go
func (p *Peer) negotiateOutboundProtocol() error {
    if err := p.writeLocalVersionMsg(); err != nil {
        return err
    }

    return p.readRemoteVersionMsg()
}
```

```go
func (p *Peer) negotiateInboundProtocol() error {
    if err := p.readRemoteVersionMsg(); err != nil {
        return err
    }

    return p.writeLocalVersionMsg()
}
```

!!!> 节点建立连接之后，发或收的第一条消息必须是version消息

### 1.3.1. 发送version消息

To connect to a known peer, nodes establish a TCP connection, usually to port 8333 (the port generally known as the one used by bitcoin), or an alternative port if one is provided. Upon establishing a connection, the node will start a "handshake" (see The initial handshake between peers) by transmitting a version message, which contains basic identifying information, including:

>1. nVersion:  
``The bitcoin P2P protocol version the client "speaks" (e.g., 70002)``
>2. nLocalServices:  
``A list of local services supported by the node, currently just NODE_NETWORK``
>3. nTime  
``The current time``
>4. addrYou  
``The IP address of the remote node as seen from this node``
>5. addrMe  
``The IP address of the local node, as discovered by the local node``
>6. subver  
``A sub-version showing the type of software running on this node (e.g., /Satoshi:0.9.2.1/)``
>7. BestHeight  
``The block height of this node’s blockchain``

可以看到，写消息是一个独立的逻辑，不依赖queueHandler等。创建一个消息之后，直接发送出去。

```go
// writeLocalVersionMsg writes our version message to the remote peer.
func (p *Peer) writeLocalVersionMsg() error {
    localVerMsg, err := p.localVersionMsg()
    if err != nil {
        return err
    }

    return p.writeMessage(localVerMsg, wire.LatestEncoding)
}
```

这里创建MsgVersion是一个重点。涉及到两个节点是否能正常沟通。

> 创建MsgVersion

```go
// localVersionMsg creates a version message that can be used to send to the
// remote peer.
func (p *Peer) localVersionMsg() (*wire.MsgVersion, error) {
    var blockNum int32
    if p.cfg.NewestBlock != nil {
        var err error
        _, blockNum, err = p.cfg.NewestBlock()
        if err != nil {
            return nil, err
        }
    }

    theirNA := p.na

    // If we are behind a proxy and the connection comes from the proxy then
    // we return an unroutable address as their address. This is to prevent
    // leaking the tor proxy address.
    if p.cfg.Proxy != "" {
        proxyaddress, _, err := net.SplitHostPort(p.cfg.Proxy)
        // invalid proxy means poorly configured, be on the safe side.
        if err != nil || p.na.IP.String() == proxyaddress {
            theirNA = wire.NewNetAddressIPPort(net.IP([]byte{0, 0, 0, 0}), 0,
                theirNA.Services)
        }
    }

    // Create a wire.NetAddress with only the services set to use as the
    // "addrme" in the version message.
    //
    // Older nodes previously added the IP and port information to the
    // address manager which proved to be unreliable as an inbound
    // connection from a peer didn't necessarily mean the peer itself
    // accepted inbound connections.
    //
    // Also, the timestamp is unused in the version message.
    ourNA := &wire.NetAddress{
        Services: p.cfg.Services,
    }

    // Generate a unique nonce for this peer so self connections can be
    // detected.  This is accomplished by adding it to a size-limited map of
    // recently seen nonces.
    nonce := uint64(rand.Int63())
    sentNonces.Add(nonce)

    // Version message.
    msg := wire.NewMsgVersion(ourNA, theirNA, nonce, blockNum)
    msg.AddUserAgent(p.cfg.UserAgentName, p.cfg.UserAgentVersion,
        p.cfg.UserAgentComments...)

    // Advertise local services.
    msg.Services = p.cfg.Services

    // Advertise our max supported protocol version.
    msg.ProtocolVersion = int32(p.cfg.ProtocolVersion)

    // Advertise if inv messages for transactions are desired.
    msg.DisableRelayTx = p.cfg.DisableRelayTx

    return msg, nil
}
```

localVersionMsg方法就是把自己节点的相关信息封装到MsgVersion中。

```go
func NewMsgVersion(me *NetAddress, you *NetAddress, nonce uint64,
	lastBlock int32) *MsgVersion {

    // Limit the timestamp to one second precision since the protocol
    // doesn't support better.
    return &MsgVersion{
        ProtocolVersion: int32(ProtocolVersion),
        Services:        0,
        Timestamp:       time.Unix(time.Now().Unix(), 0),
        AddrYou:         *you,
        AddrMe:          *me,
        Nonce:           nonce,
        UserAgent:       DefaultUserAgent,
        LastBlock:       lastBlock,
        DisableRelayTx:  false,
    }
}
```

其中，获取当前节点的区块高是调用server.newestBlock()

```go
// newestBlock returns the current best block hash and height using the format
// required by the configuration for the peer package.
func (sp *serverPeer) newestBlock() (*chainhash.Hash, int32, error) {
    best := sp.server.chain.BestSnapshot()
    return &best.Hash, best.Height, nil
}
```

### 1.3.2. 读version消息

>读版本消息也是一个独立的逻辑。

1. 读出消息内容
2. 更新当前节点peer相关状态
3. 调用OnVersion
4. 如果版本低于MinAcceptableProtocolVersion，回复reject packet，然后断开

```go
// readRemoteVersionMsg waits for the next message to arrive from the remote
// peer.  If the next message is not a version message or the version is not
// acceptable then return an error.
func (p *Peer) readRemoteVersionMsg() error {
    // Read their version message.
    remoteMsg, _, err := p.readMessage(wire.LatestEncoding)
    if err != nil {
        return err
    }

    // Notify and disconnect clients if the first message is not a version
    // message.
    msg, ok := remoteMsg.(*wire.MsgVersion)
    if !ok {
        reason := "a version message must precede all others"
        rejectMsg := wire.NewMsgReject(msg.Command(), wire.RejectMalformed,
            reason)
        _ = p.writeMessage(rejectMsg, wire.LatestEncoding)
        return errors.New(reason)
    }

    // Detect self connections.
    if !allowSelfConns && sentNonces.Exists(msg.Nonce) {
        return errors.New("disconnecting peer connected to self")
    }

    // Negotiate the protocol version and set the services to what the remote
    // peer advertised.
    p.flagsMtx.Lock()
    p.advertisedProtoVer = uint32(msg.ProtocolVersion)
    p.protocolVersion = minUint32(p.protocolVersion, p.advertisedProtoVer)
    p.versionKnown = true
    p.services = msg.Services
    p.flagsMtx.Unlock()
    log.Debugf("Negotiated protocol version %d for peer %s",
        p.protocolVersion, p)

    // Updating a bunch of stats including block based stats, and the
    // peer's time offset.
    p.statsMtx.Lock()
    p.lastBlock = msg.LastBlock
    p.startingHeight = msg.LastBlock
    p.timeOffset = msg.Timestamp.Unix() - time.Now().Unix()
    p.statsMtx.Unlock()

    // Set the peer's ID, user agent, and potentially the flag which
    // specifies the witness support is enabled.
    p.flagsMtx.Lock()
    p.id = atomic.AddInt32(&nodeCount, 1)
    p.userAgent = msg.UserAgent

    // Determine if the peer would like to receive witness data with
    // transactions, or not.
    if p.services&wire.SFNodeWitness == wire.SFNodeWitness {
        p.witnessEnabled = true
    }
    p.flagsMtx.Unlock()

    // Once the version message has been exchanged, we're able to determine
    // if this peer knows how to encode witness data over the wire
    // protocol. If so, then we'll switch to a decoding mode which is
    // prepared for the new transaction format introduced as part of
    // BIP0144.
    if p.services&wire.SFNodeWitness == wire.SFNodeWitness {
        p.wireEncoding = wire.WitnessEncoding
    }

    // Invoke the callback if specified.
    if p.cfg.Listeners.OnVersion != nil {
        rejectMsg := p.cfg.Listeners.OnVersion(p, msg)
        if rejectMsg != nil {
            _ = p.writeMessage(rejectMsg, wire.LatestEncoding)
            return errors.New(rejectMsg.Reason)
        }
    }

    // Notify and disconnect clients that have a protocol version that is
    // too old.
    //
    // NOTE: If minAcceptableProtocolVersion is raised to be higher than
    // wire.RejectVersion, this should send a reject packet before
    // disconnecting.
    if uint32(msg.ProtocolVersion) < MinAcceptableProtocolVersion {
        // Send a reject message indicating the protocol version is
        // obsolete and wait for the message to be sent before
        // disconnecting.
        reason := fmt.Sprintf("protocol version must be %d or greater",
            MinAcceptableProtocolVersion)
        rejectMsg := wire.NewMsgReject(msg.Command(), wire.RejectObsolete,
            reason)
        _ = p.writeMessage(rejectMsg, wire.LatestEncoding)
        return errors.New(reason)
    }

    return nil
}
```

>OnVersion是个很重要的方法，我们来看看

```go
// OnVersion is invoked when a peer receives a version bitcoin message
// and is used to negotiate the protocol version details as well as kick start
// the communications.
func (sp *serverPeer) OnVersion(_ *peer.Peer, msg *wire.MsgVersion) *wire.MsgReject {
    // Update the address manager with the advertised services for outbound
    // connections in case they have changed.  This is not done for inbound
    // connections to help prevent malicious behavior and is skipped when
    // running on the simulation test network since it is only intended to
    // connect to specified peers and actively avoids advertising and
    // connecting to discovered peers.
    //
    // NOTE: This is done before rejecting peers that are too old to ensure
    // it is updated regardless in the case a new minimum protocol version is
    // enforced and the remote node has not upgraded yet.
    isInbound := sp.Inbound()
    remoteAddr := sp.NA()
    addrManager := sp.server.addrManager
    if !cfg.SimNet && !isInbound {
        addrManager.SetServices(remoteAddr, msg.Services)
    }

    // Ignore peers that have a protcol version that is too old.  The peer
    // negotiation logic will disconnect it after this callback returns.
    if msg.ProtocolVersion < int32(peer.MinAcceptableProtocolVersion) {
        return nil
    }

    // Reject outbound peers that are not full nodes.
    wantServices := wire.SFNodeNetwork
    if !isInbound && !hasServices(msg.Services, wantServices) {
        missingServices := wantServices & ^msg.Services
        srvrLog.Debugf("Rejecting peer %s with services %v due to not "+
            "providing desired services %v", sp.Peer, msg.Services,
            missingServices)
        reason := fmt.Sprintf("required services %#x not offered",
            uint64(missingServices))
        return wire.NewMsgReject(msg.Command(), wire.RejectNonstandard, reason)
    }

    // Update the address manager and request known addresses from the
    // remote peer for outbound connections.  This is skipped when running
    // on the simulation test network since it is only intended to connect
    // to specified peers and actively avoids advertising and connecting to
    // discovered peers.
    if !cfg.SimNet && !isInbound {
        // After soft-fork activation, only make outbound
        // connection to peers if they flag that they're segwit
        // enabled.
        chain := sp.server.chain
        segwitActive, err := chain.IsDeploymentActive(chaincfg.DeploymentSegwit)
        if err != nil {
            peerLog.Errorf("Unable to query for segwit soft-fork state: %v",
                err)
            return nil
        }

        if segwitActive && !sp.IsWitnessEnabled() {
            peerLog.Infof("Disconnecting non-segwit peer %v, isn't segwit "+
                "enabled and we need more segwit enabled peers", sp)
            sp.Disconnect()
            return nil
        }

        // Advertise the local address when the server accepts incoming
        // connections and it believes itself to be close to the best known tip.
        if !cfg.DisableListen && sp.server.syncManager.IsCurrent() {
            // Get address that best matches.
            lna := addrManager.GetBestLocalAddress(remoteAddr)
            if addrmgr.IsRoutable(lna) {
                // Filter addresses the peer already knows about.
                addresses := []*wire.NetAddress{lna}
                sp.pushAddrMsg(addresses)
            }
        }

        // Request known addresses if the server address manager needs
        // more and the peer has a protocol version new enough to
        // include a timestamp with addresses.
        hasTimestamp := sp.ProtocolVersion() >= wire.NetAddressTimeVersion
        if addrManager.NeedMoreAddresses() && hasTimestamp {
            sp.QueueMessage(wire.NewMsgGetAddr(), nil)
        }

        // Mark the address as a known good address.
        addrManager.Good(remoteAddr)
    }

    // Add the remote peer time as a sample for creating an offset against
    // the local clock to keep the network time in sync.
    sp.server.timeSource.AddTimeSample(sp.Addr(), msg.Timestamp)

    // Signal the sync manager this peer is a new sync candidate.
    sp.server.syncManager.NewPeer(sp.Peer)

    // Choose whether or not to relay transactions before a filter command
    // is received.
    sp.setDisableRelayTx(msg.DisableRelayTx)

    // Add valid peer to the server.
    sp.server.AddPeer(sp)
    return nil
}
```

因为出去或者进来的连接都会调用此方法，因此，上面会根据isInbound做些判断处理。比如，主动连结过去的节点如果没有SFNodeNetwork服务，就会回复MsgReject。这个回调处理，可以判断出节点的一些状态。因此干的事比较多。

1. 更新地址拥有的服务
```go
addrManager.SetServices(remoteAddr, msg.Services)
```
2. 回复最优的本地地址给对方
```go
lna := addrManager.GetBestLocalAddress(remoteAddr)
if addrmgr.IsRoutable(lna) {
    // Filter addresses the peer already knows about.
    addresses := []*wire.NetAddress{lna}
    sp.pushAddrMsg(addresses)
}
```
3. 向对方节点发送GetAddr消息,获取更多地址
```go
sp.QueueMessage(wire.NewMsgGetAddr(), nil)
```
4. 标记当前地址是可用的。
```go
addrManager.Good(remoteAddr)
```
5. 添加时间样本
```go
// Add the remote peer time as a sample for creating an offset against
// the local clock to keep the network time in sync.
sp.server.timeSource.AddTimeSample(sp.Addr(), msg.Timestamp)
```
6. 同步管理器中可以添加一个节用的节点
```go
sp.server.syncManager.NewPeer(sp.Peer)
```
6. 添加一个可用的节点到server中
```go
sp.server.AddPeer(sp)
```

>其中，addrManager.Good(remoteAddr)可以影响前面章节提到过的addrmanager.GetAddress()中地址权重。

```go
// Good marks the given address as good.  To be called after a successful
// connection and version exchange.  If the address is unknown to the address
// manager it will be ignored.
func (a *AddrManager) Good(addr *wire.NetAddress) {
    a.mtx.Lock()
    defer a.mtx.Unlock()

    ka := a.find(addr)
    if ka == nil {
        return
    }

    // ka.Timestamp is not updated here to avoid leaking information
    // about currently connected peers.
    now := time.Now()
    ka.lastsuccess = now
    ka.lastattempt = now
    ka.attempts = 0

    // move to tried set, optionally evicting other addresses if neeed.
    if ka.tried {
        return
    }

    // ok, need to move it to tried.

    // remove from all new buckets.
    // record one of the buckets in question and call it the `first'
    addrKey := NetAddressKey(addr)
    oldBucket := -1
    for i := range a.addrNew {
        // we check for existence so we can record the first one
        if _, ok := a.addrNew[i][addrKey]; ok {
            delete(a.addrNew[i], addrKey)
            ka.refs--
            if oldBucket == -1 {
                oldBucket = i
            }
        }
    }
    a.nNew--

    if oldBucket == -1 {
        // What? wasn't in a bucket after all.... Panic?
        return
    }

    bucket := a.getTriedBucket(ka.na)

    // Room in this tried bucket?
    if a.addrTried[bucket].Len() < triedBucketSize {
        ka.tried = true
        a.addrTried[bucket].PushBack(ka)
        a.nTried++
        return
    }

    // No room, we have to evict something else.
    entry := a.pickTried(bucket)
    rmka := entry.Value.(*KnownAddress)

    // First bucket it would have been put in.
    newBucket := a.getNewBucket(rmka.na, rmka.srcAddr)

    // If no room in the original bucket, we put it in a bucket we just
    // freed up a space in.
    if len(a.addrNew[newBucket]) >= newBucketSize {
        newBucket = oldBucket
    }

    // replace with ka in list.
    ka.tried = true
    entry.Value = ka

    rmka.tried = false
    rmka.refs++

    // We don't touch a.nTried here since the number of tried stays the same
    // but we decemented new above, raise it again since we're putting
    // something back.
    a.nNew++

    rmkey := NetAddressKey(rmka.na)
    log.Tracef("Replacing %s with %s in tried", rmkey, addrKey)

    // We made sure there is space here just above.
    a.addrNew[newBucket][rmkey] = rmka
}
```

> 可以看到，它会对这个节点的地址信息做处理，提高这个地址被取到的机会。

```go
now := time.Now()
ka.lastsuccess = now
ka.lastattempt = now
ka.attempts = 0
```

## 1.4. 消息收发

在上面，我们看到调用一个serverpeer.QueueMessage发送了一条wire.NewMsgGetAddr()消息。我们回到这里，根着这个流程，看看消息收发的流程，内部逻辑。

### 1.4.1. 消息结构

>所有消息都实现Message接口的方法,每个消息都有一个command,用于识别。先看下Message接口和MsgGetAddr消息

```go

// Message is an interface that describes a bitcoin message.  A type that
// implements Message has complete control over the representation of its data
// and may therefore contain additional or fewer fields than those which
// are used directly in the protocol encoded message.
type Message interface {
    BtcDecode(io.Reader, uint32, MessageEncoding) error
    BtcEncode(io.Writer, uint32, MessageEncoding) error
    Command() string
    MaxPayloadLength(uint32) uint32
}

// MsgGetAddr implements the Message interface and represents a bitcoin
// getaddr message.  It is used to request a list of known active peers on the
// network from a peer to help identify potential nodes.  The list is returned
// via one or more addr messages (MsgAddr).
//
// This message has no payload.
type MsgGetAddr struct{}

// Command returns the protocol command string for the message.  This is part
// of the Message interface implementation.
func (msg *MsgGetAddr) Command() string {
    return CmdGetAddr
}

// CmdGetAddr = "getaddr"
```

>**全部消息命令**：

```go
// MaxMessagePayload is the maximum bytes a message can be regardless of other
// individual limits imposed by messages themselves.
const MaxMessagePayload = (1024 * 1024 * 32) // 32MB

// Commands used in bitcoin message headers which describe the type of message.
const (
    CmdVersion      = "version"
    CmdVerAck       = "verack"
    CmdGetAddr      = "getaddr"
    CmdAddr         = "addr"
    CmdGetBlocks    = "getblocks"
    CmdInv          = "inv"
    CmdGetData      = "getdata"
    CmdNotFound     = "notfound"
    CmdBlock        = "block"
    CmdTx           = "tx"
    CmdGetHeaders   = "getheaders"
    CmdHeaders      = "headers"
    CmdPing         = "ping"
    CmdPong         = "pong"
    CmdAlert        = "alert"
    CmdMemPool      = "mempool"
    CmdFilterAdd    = "filteradd"
    CmdFilterClear  = "filterclear"
    CmdFilterLoad   = "filterload"
    CmdMerkleBlock  = "merkleblock"
    CmdReject       = "reject"
    CmdSendHeaders  = "sendheaders"
    CmdFeeFilter    = "feefilter"
    CmdGetCFilters  = "getcfilters"
    CmdGetCFHeaders = "getcfheaders"
    CmdGetCFCheckpt = "getcfcheckpt"
    CmdCFilter      = "cfilter"
    CmdCFHeaders    = "cfheaders"
    CmdCFCheckpt    = "cfcheckpt"
)
```


### 1.4.2. 发消息

>QueueMessage为peer提供的消息发送接口,会有如下几步

#### 1.4.2.1. 写outputQueue

```go
// QueueMessage adds the passed bitcoin message to the peer send queue.
//
// This function is safe for concurrent access.
func (p *Peer) QueueMessage(msg wire.Message, doneChan chan<- struct{}) {
    p.QueueMessageWithEncoding(msg, doneChan, wire.BaseEncoding)
}

// QueueMessageWithEncoding adds the passed bitcoin message to the peer send
// queue. This function is identical to QueueMessage, however it allows the
// caller to specify the wire encoding type that should be used when
// encoding/decoding blocks and transactions.
//
// This function is safe for concurrent access.
func (p *Peer) QueueMessageWithEncoding(msg wire.Message, doneChan chan<- struct{},
    encoding wire.MessageEncoding) {

    // Avoid risk of deadlock if goroutine already exited.  The goroutine
    // we will be sending to hangs around until it knows for a fact that
    // it is marked as disconnected and *then* it drains the channels.
    if !p.Connected() {
        if doneChan != nil {
            go func() {
                doneChan <- struct{}{}
            }()
        }
        return
    }
    p.outputQueue <- outMsg{msg: msg, encoding: encoding, doneChan: doneChan}
}
```

#### 1.4.2.2. queueHandler

>最终会创建一个outMsg放到outputQueue中。而outputQueue是在**queueHandler**中处理的。

```go
// queueHandler handles the queuing of outgoing data for the peer. This runs as
// a muxer for various sources of input so we can ensure that server and peer
// handlers will not block on us sending a message.  That data is then passed on
// to outHandler to be actually written.
func (p *Peer) queueHandler() {
    pendingMsgs := list.New()
    invSendQueue := list.New()
    trickleTicker := time.NewTicker(p.cfg.TrickleInterval)
    defer trickleTicker.Stop()

    // We keep the waiting flag so that we know if we have a message queued
    // to the outHandler or not.  We could use the presence of a head of
    // the list for this but then we have rather racy concerns about whether
    // it has gotten it at cleanup time - and thus who sends on the
    // message's done channel.  To avoid such confusion we keep a different
    // flag and pendingMsgs only contains messages that we have not yet
    // passed to outHandler.
    waiting := false

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
out:
    for {
        select {
        case msg := <-p.outputQueue:
            waiting = queuePacket(msg, pendingMsgs, waiting)
        // This channel is notified when a message has been sent across
        // the network socket.
        case <-p.sendDoneQueue:
            // No longer waiting if there are no more messages
            // in the pending messages queue.
            next := pendingMsgs.Front()
            if next == nil {
                waiting = false
                continue
            }

            // Notify the outHandler about the next item to
            // asynchronously send.
            val := pendingMsgs.Remove(next)
            p.sendQueue <- val.(outMsg)
        ...
        }
    }
    ...
}
```

通过调用queuePacket，消息最后还是推到sendQueue。这里封装了一下，当waiting=false 时，直接发送，否则加才到pendingMsgs中，收到发送完成的通知之后，再去pendingMsgs取一下个消息发送。

#### 1.4.2.3. outHandler

```go
func (p *Peer) outHandler() {
out:
    for {
        select {
        case msg := <-p.sendQueue:
            ... ping 处理

            p.stallControl <- stallControlMsg{sccSendMessage, msg.msg}

            err := p.writeMessage(msg.msg, msg.encoding)
            if err != nil {
                p.Disconnect()
                if p.shouldLogWriteError(err) {
                    log.Errorf("Failed to send message to "+
                        "%s: %v", p, err)
                }
                if msg.doneChan != nil {
                    msg.doneChan <- struct{}{}
                }
                continue
            }

            // At this point, the message was successfully sent, so
            // update the last send time, signal the sender of the
            // message that it has been sent (if requested), and
            // signal the send queue to the deliver the next queued
            // message.
            atomic.StoreInt64(&p.lastSend, time.Now().Unix())
            if msg.doneChan != nil {
                msg.doneChan <- struct{}{}
            }
            p.sendDoneQueue <- struct{}{}

        case <-p.quit:
            break out
        }
    }
    ...
}
```

> 这里，做了几件事：
>1. 发送一个sccSendMessage通知给stallHandler
>2. 发送消息到socket
>3. lastSend重置
>4. 写doneChan，唤醒等待的goroutine
>5. 写p.sendDoneQueue，通知queueHandler处理缓冲队列中其它消息

#### 1.4.2.4. 消息发送到节点

```go
// writeMessage sends a bitcoin message to the peer with logging.
func (p *Peer) writeMessage(msg wire.Message, enc wire.MessageEncoding) error {
    // Don't do anything if we're disconnecting.
    if atomic.LoadInt32(&p.disconnect) != 0 {
        return nil
    }

    ... log info

    // Write the message to the peer.
    n, err := wire.WriteMessageWithEncodingN(p.conn, msg,
        p.ProtocolVersion(), p.cfg.ChainParams.Net, enc)
    atomic.AddUint64(&p.bytesSent, uint64(n))
    if p.cfg.Listeners.OnWrite != nil {
        p.cfg.Listeners.OnWrite(p, n, msg, err)
    }
    return err
}
```

- wire.WriteMessageWithEncodingN： 真正写消息到socket的方法。
- server.OnWrite： 用于通知server当前发送的数据量

```go
// OnWrite is invoked when a peer sends a message and it is used to update
// the bytes sent by the server.
func (sp *serverPeer) OnWrite(_ *peer.Peer, bytesWritten int, msg wire.Message, err error) {
    sp.server.AddBytesSent(uint64(bytesWritten))
}
```

### 1.4.3. 收到消息

当一个节点A连结到节点B，并发送了一个请求时，B节点收到消息要处理。在Peer.Start()中启动的inHandler()，就是用于接收消息并处理的。我们以上面发的MsgGetAddr为例看下收到消息之后的处理。

#### 1.4.3.1. inHandler

```go
func (p *Peer) inHandler() {
    // The timer is stopped when a new message is received and reset after it
    // is processed.
    idleTimer := time.AfterFunc(idleTimeout, func() {
        log.Warnf("Peer %s no answer for %s -- disconnecting", p, idleTimeout)
        p.Disconnect()
    })

out:
    for atomic.LoadInt32(&p.disconnect) == 0 {
        // Read a message and stop the idle timer as soon as the read
        // is done.  The timer is reset below for the next iteration if
        // needed.
        rmsg, buf, err := p.readMessage(p.wireEncoding)
        idleTimer.Stop()
        if err != nil {
           ... error deal
        }
        atomic.StoreInt64(&p.lastRecv, time.Now().Unix())
        p.stallControl <- stallControlMsg{sccReceiveMessage, rmsg}

        // Handle each supported message type.
        p.stallControl <- stallControlMsg{sccHandlerStart, rmsg}
        switch msg := rmsg.(type) {
        case *wire.MsgVersion:
            // Limit to one version message per peer.
            p.PushRejectMsg(msg.Command(), wire.RejectDuplicate,
                "duplicate version message", nil, true)
            break out

        case *wire.MsgVerAck:

            // No read lock is necessary because verAckReceived is not written
            // to in any other goroutine.
            if p.verAckReceived {
                log.Infof("Already received 'verack' from peer %v -- "+
                    "disconnecting", p)
                break out
            }
            p.flagsMtx.Lock()
            p.verAckReceived = true
            p.flagsMtx.Unlock()
            if p.cfg.Listeners.OnVerAck != nil {
                p.cfg.Listeners.OnVerAck(p, msg)
            }
        case *wire.MsgGetAddr:
            if p.cfg.Listeners.OnGetAddr != nil {
                p.cfg.Listeners.OnGetAddr(p, msg)
            }

        ...ignore more than MsgGetAddr/MsgAddr/MsgPing/MsgPong..

        default:
            log.Debugf("Received unhandled message of type %v "+
                "from %v", rmsg.Command(), p)
        }
        p.stallControl <- stallControlMsg{sccHandlerDone, rmsg}

        // A message was received so reset the idle timer.
        idleTimer.Reset(idleTimeout)
    }

    // Ensure the idle timer is stopped to avoid leaking the resource.
    idleTimer.Stop()

    // Ensure connection is closed.
    p.Disconnect()

    close(p.inQuit)
    log.Tracef("Peer input handler done for %s", p)
}

```

这个handler中，只要没有断开连接，会一直循环读消息，上面使用idleTimer实现了超时处理，超过5分钟没有收到消息就会调用回调方法，断开此节点的连接。跳出循环。
每次会从节点读取一条消息，我们看看读消息方法

```go
// readMessage reads the next bitcoin message from the peer with logging.
func (p *Peer) readMessage(encoding wire.MessageEncoding) (wire.Message, []byte, error) {
    n, msg, buf, err := wire.ReadMessageWithEncodingN(p.conn,
        p.ProtocolVersion(), p.cfg.ChainParams.Net, encoding)
    atomic.AddUint64(&p.bytesReceived, uint64(n))
    if p.cfg.Listeners.OnRead != nil {
        p.cfg.Listeners.OnRead(p, n, msg, err)
    }
    if err != nil {
        return nil, nil, err
    }

    // Use closures to log expensive operations so they are only run when
    // the logging level requires it.
    log.Debugf("%v", newLogClosure(func() string {
        // Debug summary of message.
        summary := messageSummary(msg)
        if len(summary) > 0 {
            summary = " (" + summary + ")"
        }
        return fmt.Sprintf("Received %v%s from %s",
            msg.Command(), summary, p)
    }))
    log.Tracef("%v", newLogClosure(func() string {
        return spew.Sdump(msg)
    }))
    log.Tracef("%v", newLogClosure(func() string {
        return spew.Sdump(buf)
    }))

    return msg, buf, nil
}
```

#### 1.4.3.2. 消息处理

>几乎所有消息的处理最后都是回调到server.go中ServerPeer结构体中的方法了。我们看下OnGetAddr这个回调的实现

```go

// OnGetAddr is invoked when a peer receives a getaddr bitcoin message
// and is used to provide the peer with known addresses from the address
// manager.
func (sp *serverPeer) OnGetAddr(_ *peer.Peer, msg *wire.MsgGetAddr) {
    // Don't return any addresses when running on the simulation test
    // network.  This helps prevent the network from becoming another
    // public test network since it will not be able to learn about other
    // peers that have not specifically been provided.
    if cfg.SimNet {
        return
    }

    // Do not accept getaddr requests from outbound peers.  This reduces
    // fingerprinting attacks.
    if !sp.Inbound() {
        peerLog.Debugf("Ignoring getaddr request from outbound peer ",
            "%v", sp)
        return
    }

    // Only allow one getaddr request per connection to discourage
    // address stamping of inv announcements.
    if sp.sentAddrs {
        peerLog.Debugf("Ignoring repeated getaddr request from peer ",
            "%v", sp)
        return
    }
    sp.sentAddrs = true

    // Get the current known addresses from the address manager.
    addrCache := sp.server.addrManager.AddressCache()

    // Push the addresses.
    sp.pushAddrMsg(addrCache)
}
```

#### 1.4.3.3. 消息回复

最后两行代码，从地址管理器中得到可用的地址列表，调用pushAddrMsg返回。

```go
// pushAddrMsg sends an addr message to the connected peer using the provided
// addresses.
func (sp *serverPeer) pushAddrMsg(addresses []*wire.NetAddress) {
    // Filter addresses already known to the peer.
    addrs := make([]*wire.NetAddress, 0, len(addresses))
    for _, addr := range addresses {
        if !sp.addressKnown(addr) {
            addrs = append(addrs, addr)
        }
    }
    known, err := sp.PushAddrMsg(addrs)
    if err != nil {
        peerLog.Errorf("Can't push address message to %s: %v", sp.Peer, err)
        sp.Disconnect()
        return
    }
    sp.addKnownAddresses(known)
}
```

>serverPeer中会调用peer中的同名方法

```go
// PushAddrMsg sends an addr message to the connected peer using the provided
// addresses.  This function is useful over manually sending the message via
// QueueMessage since it automatically limits the addresses to the maximum
// number allowed by the message and randomizes the chosen addresses when there
// are too many.  It returns the addresses that were actually sent and no
// message will be sent if there are no entries in the provided addresses slice.
//
// This function is safe for concurrent access.
func (p *Peer) PushAddrMsg(addresses []*wire.NetAddress) ([]*wire.NetAddress, error) {
    addressCount := len(addresses)

    // Nothing to send.
    if addressCount == 0 {
        return nil, nil
    }

    msg := wire.NewMsgAddr()
    msg.AddrList = make([]*wire.NetAddress, addressCount)
    copy(msg.AddrList, addresses)

    // Randomize the addresses sent if there are more than the maximum allowed.
    if addressCount > wire.MaxAddrPerMsg {
        // Shuffle the address list.
        for i := 0; i < wire.MaxAddrPerMsg; i++ {
            j := i + rand.Intn(addressCount-i)
            msg.AddrList[i], msg.AddrList[j] = msg.AddrList[j], msg.AddrList[i]
        }

        // Truncate it to the maximum size.
        msg.AddrList = msg.AddrList[:wire.MaxAddrPerMsg]
    }

    p.QueueMessage(msg, nil)
    return msg.AddrList, nil
}
```

>这个方法又会生成一个MsgAddr消息体，这个消息有消息内容AddrList，因此，会有相应的编码/解码实现。调用QueueMessage把消息发送给A节点之后，在A节点的inhandler会有对应的处理方法。

```go
// NewMsgAddr returns a new bitcoin addr message that conforms to the
// Message interface.  See MsgAddr for details.
func NewMsgAddr() *MsgAddr {
    return &MsgAddr{
        AddrList: make([]*NetAddress, 0, MaxAddrPerMsg),
    }
}
```

>msgaddr.go

```go
const MaxAddrPerMsg = 1000

// MsgAddr implements the Message interface and represents a bitcoin
// addr message.  It is used to provide a list of known active peers on the
// network.  An active peer is considered one that has transmitted a message
// within the last 3 hours.  Nodes which have not transmitted in that time
// frame should be forgotten.  Each message is limited to a maximum number of
// addresses, which is currently 1000.  As a result, multiple messages must
// be used to relay the full list.
//
// Use the AddAddress function to build up the list of known addresses when
// sending an addr message to another peer.
type MsgAddr struct {
    AddrList []*NetAddress
}

// AddAddress adds a known active peer to the message.
func (msg *MsgAddr) AddAddress(na *NetAddress) error {
    if len(msg.AddrList)+1 > MaxAddrPerMsg {
        str := fmt.Sprintf("too many addresses in message [max %v]",
            MaxAddrPerMsg)
        return messageError("MsgAddr.AddAddress", str)
    }

    msg.AddrList = append(msg.AddrList, na)
    return nil
}

// AddAddresses adds multiple known active peers to the message.
func (msg *MsgAddr) AddAddresses(netAddrs ...*NetAddress) error {
    for _, na := range netAddrs {
        err := msg.AddAddress(na)
        if err != nil {
            return err
        }
    }
    return nil
}

// ClearAddresses removes all addresses from the message.
func (msg *MsgAddr) ClearAddresses() {
    msg.AddrList = []*NetAddress{}
}

// BtcDecode decodes r using the bitcoin protocol encoding into the receiver.
// This is part of the Message interface implementation.
func (msg *MsgAddr) BtcDecode(r io.Reader, pver uint32, enc MessageEncoding) error {
    count, err := ReadVarInt(r, pver)
    if err != nil {
        return err
    }

    // Limit to max addresses per message.
    if count > MaxAddrPerMsg {
        str := fmt.Sprintf("too many addresses for message "+
            "[count %v, max %v]", count, MaxAddrPerMsg)
        return messageError("MsgAddr.BtcDecode", str)
    }

    addrList := make([]NetAddress, count)
    msg.AddrList = make([]*NetAddress, 0, count)
    for i := uint64(0); i < count; i++ {
        na := &addrList[i]
        err := readNetAddress(r, pver, na, true)
        if err != nil {
            return err
        }
        msg.AddAddress(na)
    }
    return nil
}

// BtcEncode encodes the receiver to w using the bitcoin protocol encoding.
// This is part of the Message interface implementation.
func (msg *MsgAddr) BtcEncode(w io.Writer, pver uint32, enc MessageEncoding) error {
    // Protocol versions before MultipleAddressVersion only allowed 1 address
    // per message.
    count := len(msg.AddrList)
    if pver < MultipleAddressVersion && count > 1 {
        str := fmt.Sprintf("too many addresses for message of "+
            "protocol version %v [count %v, max 1]", pver, count)
        return messageError("MsgAddr.BtcEncode", str)

    }
    if count > MaxAddrPerMsg {
        str := fmt.Sprintf("too many addresses for message "+
            "[count %v, max %v]", count, MaxAddrPerMsg)
        return messageError("MsgAddr.BtcEncode", str)
    }

    err := WriteVarInt(w, pver, uint64(count))
    if err != nil {
        return err
    }

    for _, na := range msg.AddrList {
        err = writeNetAddress(w, pver, na, true)
        if err != nil {
            return err
        }
    }

    return nil
}

// Command returns the protocol command string for the message.  This is part
// of the Message interface implementation.
func (msg *MsgAddr) Command() string {
    return CmdAddr
}

// MaxPayloadLength returns the maximum length the payload can be for the
// receiver.  This is part of the Message interface implementation.
func (msg *MsgAddr) MaxPayloadLength(pver uint32) uint32 {
    if pver < MultipleAddressVersion {
        // Num addresses (varInt) + a single net addresses.
        return MaxVarIntPayload + maxNetAddressPayload(pver)
    }

    // Num addresses (varInt) + max allowed addresses.
    return MaxVarIntPayload + (MaxAddrPerMsg * maxNetAddressPayload(pver))
}

// NewMsgAddr returns a new bitcoin addr message that conforms to the
// Message interface.  See MsgAddr for details.
func NewMsgAddr() *MsgAddr {
    return &MsgAddr{
        AddrList: make([]*NetAddress, 0, MaxAddrPerMsg),
    }
}
```

## 1.5. 心跳机制

心跳机制是定时发送一个自定义的结构体(心跳包)，让对方知道自己还活着，以确保连接的有效性的机制。
在TCP的机制里面，本身是存在有心跳包的机制的，也就是TCP的选项。系统默认是设置的是2小时的心跳频率。但是它检查不到机器断电、网线拔出、防火墙这些断线。而且逻辑层处理断线可能也不是那么好处理。一般，如果只是用于保活还是可以的。心跳包一般来说都是在逻辑层发送空的包来实现的。在长连接下，有可能很长一段时间都没有数据往来。理论上说，这个连接是一直保持连接的，但是实际情况中，如果中间节点出现什么故障是难以知道的。更要命的是，有的节点（防火墙）会自动把一定时间之内没有数据交互的连接给断掉。在这个时候，就需要我们的心跳包了，用于维持长连接，保活。

在btcd中，pingHandler就是用于心跳包发送

### 1.5.1. pingHandler

```go
// pingHandler periodically pings the peer.  It must be run as a goroutine.
func (p *Peer) pingHandler() {
    pingTicker := time.NewTicker(pingInterval)
    defer pingTicker.Stop()

out:
    for {
        select {
        case <-pingTicker.C:
            nonce, err := wire.RandomUint64()
            if err != nil {
                log.Errorf("Not sending ping to %s: %v", p, err)
                continue
            }
            p.QueueMessage(wire.NewMsgPing(nonce), nil)

        case <-p.quit:
            break out
        }
    }
}
```

### 1.5.2. 处理心跳

>消息处理都在peer.inHandler中，我们看下ping的处理

```go
case *wire.MsgPing:
    p.handlePingMsg(msg)
    if p.cfg.Listeners.OnPing != nil {
        p.cfg.Listeners.OnPing(p, msg)
    }
```

>不管上层serverPeer有没有实现ping处理，在peer层都会处理，是实上上层也没有任何处理操作。

```go
// handlePingMsg is invoked when a peer receives a ping bitcoin message.  For
// recent clients (protocol version > BIP0031Version), it replies with a pong
// message.  For older clients, it does nothing and anything other than failure
// is considered a successful ping.
func (p *Peer) handlePingMsg(msg *wire.MsgPing) {
    // Only reply with pong if the message is from a new enough client.
    if p.ProtocolVersion() > wire.BIP0031Version {
        // Include nonce from ping so pong can be identified.
        p.QueueMessage(wire.NewMsgPong(msg.Nonce), nil)
    }
}
```

pong中回复了ping过来的Nonce。

## 1.6. 消息间隔处理

当发送一个请求消息之后，希望在一定时间之内得到回复，否则就要做相关的处理。这个功能就是在stallHandler中实现的。在前面的消息收发时，我们可以看到代码中都会有写stallControl的代码。因此，在这个stallHandler中会启15秒的stallTicker，定时去处理超时的消息，断开这个节点的连接。

### 1.6.1. 写stallControl

#### 1.6.1.1. 发消息时

发消息时，会写一个sccSendMessage到stallControl。

```go
func (p *Peer) outHandler() {
out:
    for {
        select {
        case msg := <-p.sendQueue:
            switch m := msg.msg.(type) {
            case *wire.MsgPing:
                // Only expects a pong message in later protocol
                // versions.  Also set up statistics.
                if p.ProtocolVersion() > wire.BIP0031Version {
                    p.statsMtx.Lock()
                    p.lastPingNonce = m.Nonce
                    p.lastPingTime = time.Now()
                    p.statsMtx.Unlock()
                }
            }

            p.stallControl <- stallControlMsg{sccSendMessage, msg.msg}
```

#### 1.6.1.2. 收到消息时

处理消息时会有三个sccReceiveMessage，sccHandlerStart和sccHandlerDone。

```go
// inHandler handles all incoming messages for the peer.  It must be run as a
// goroutine.
func (p *Peer) inHandler() {
    // The timer is stopped when a new message is received and reset after it
    // is processed.
    idleTimer := time.AfterFunc(idleTimeout, func() {
        log.Warnf("Peer %s no answer for %s -- disconnecting", p, idleTimeout)
        p.Disconnect()
    })

    out:
    for atomic.LoadInt32(&p.disconnect) == 0 {
        // Read a message and stop the idle timer as soon as the read
        // is done.  The timer is reset below for the next iteration if
        // needed.
        rmsg, buf, err := p.readMessage(p.wireEncoding)
        idleTimer.Stop()
        ...
        atomic.StoreInt64(&p.lastRecv, time.Now().Unix())

        p.stallControl <- stallControlMsg{sccReceiveMessage, rmsg}

        // Handle each supported message type.
        p.stallControl <- stallControlMsg{sccHandlerStart, rmsg}

        switch msg := rmsg.(type) {
        ...
        }
        p.stallControl <- stallControlMsg{sccHandlerDone, rmsg}
    }
}
```

### 1.6.2. stallHandler

从A节点出发 ，当它向B节点发送一个消息之前 ，会写sccSendMessage到stallControl。当B节点回复一个消息，进入A的inHandler之后，会把sccReceiveMessage和sccHandlerStart写到stallControl，当处理完成之后，会把sccHandlerDone写到stallControl，完成一个闭环。如果没有收到对应的消息回复，stallHandler中就可以检测出来。

```go
// stallHandler handles stall detection for the peer.  This entails keeping
// track of expected responses and assigning them deadlines while accounting for
// the time spent in callbacks.  It must be run as a goroutine.
func (p *Peer) stallHandler() {
    // These variables are used to adjust the deadline times forward by the
    // time it takes callbacks to execute.  This is done because new
    // messages aren't read until the previous one is finished processing
    // (which includes callbacks), so the deadline for receiving a response
    // for a given message must account for the processing time as well.
    var handlerActive bool
    var handlersStartTime time.Time
    var deadlineOffset time.Duration

    // pendingResponses tracks the expected response deadline times.
    pendingResponses := make(map[string]time.Time)

    // stallTicker is used to periodically check pending responses that have
    // exceeded the expected deadline and disconnect the peer due to
    // stalling.
    stallTicker := time.NewTicker(stallTickInterval)
    defer stallTicker.Stop()

    // ioStopped is used to detect when both the input and output handler
    // goroutines are done.
    var ioStopped bool
out:
    for {
        select {
        case msg := <-p.stallControl:
            switch msg.command {
            case sccSendMessage:
                // Add a deadline for the expected response
                // message if needed.
                p.maybeAddDeadline(pendingResponses,
                    msg.message.Command())

            case sccReceiveMessage:
                // Remove received messages from the expected
                // response map.  Since certain commands expect
                // one of a group of responses, remove
                // everything in the expected group accordingly.
                switch msgCmd := msg.message.Command(); msgCmd {
                case wire.CmdBlock:
                    fallthrough
                case wire.CmdMerkleBlock:
                    fallthrough
                case wire.CmdTx:
                    fallthrough
                case wire.CmdNotFound:
                    delete(pendingResponses, wire.CmdBlock)
                    delete(pendingResponses, wire.CmdMerkleBlock)
                    delete(pendingResponses, wire.CmdTx)
                    delete(pendingResponses, wire.CmdNotFound)

                default:
                    delete(pendingResponses, msgCmd)
                }

            case sccHandlerStart:
                // Warn on unbalanced callback signalling.
                if handlerActive {
                    log.Warn("Received handler start " +
                        "control command while a " +
                        "handler is already active")
                    continue
                }

                handlerActive = true
                handlersStartTime = time.Now()

            case sccHandlerDone:
                // Warn on unbalanced callback signalling.
                if !handlerActive {
                    log.Warn("Received handler done " +
                        "control command when a " +
                        "handler is not already active")
                    continue
                }

                // Extend active deadlines by the time it took
                // to execute the callback.
                duration := time.Since(handlersStartTime)
                deadlineOffset += duration
                handlerActive = false

            default:
                log.Warnf("Unsupported message command %v",
                    msg.command)
            }

        case <-stallTicker.C:
            // Calculate the offset to apply to the deadline based
            // on how long the handlers have taken to execute since
            // the last tick.
            now := time.Now()
            offset := deadlineOffset
            if handlerActive {
                offset += now.Sub(handlersStartTime)
            }

            // Disconnect the peer if any of the pending responses
            // don't arrive by their adjusted deadline.
            for command, deadline := range pendingResponses {
                if now.Before(deadline.Add(offset)) {
                    continue
                }

                log.Debugf("Peer %s appears to be stalled or "+
                    "misbehaving, %s timeout -- "+
                    "disconnecting", p, command)
                p.Disconnect()
                break
            }

            // Reset the deadline offset for the next tick.
            deadlineOffset = 0

        case <-p.inQuit:
            // The stall handler can exit once both the input and
            // output handler goroutines are done.
            if ioStopped {
                break out
            }
            ioStopped = true

        case <-p.outQuit:
            // The stall handler can exit once both the input and
            // output handler goroutines are done.
            if ioStopped {
                break out
            }
            ioStopped = true
        }
    }

    // Drain any wait channels before going away so there is nothing left
    // waiting on this goroutine.
cleanup:
    for {
        select {
        case <-p.stallControl:
        default:
            break cleanup
        }
    }
    log.Tracef("Peer stall handler done for %s", p)
}
```

1. 当收到sccSendMessage时，说明发送了一条消息，这时要记录到pendingResponses中。key为command, value为时间。其中不同的命令，处理的时间不同，因此，长时间处理的命令要加长超时时间。见下面的maybeAddDeadline()
2. 当收到sccReceiveMessage时，说明消息已经收到，这里就可以删除pendingResponses中对应的命令。

>**由于当前节点可能会在发送消息之后，收到消息处理回复之前，会有其它节点请求处理，影响了消息的回复时间间隔。因此，在sccHandlerStart和sccHandlerDone中会计算时间偏移量。最后在stallTicker处理时，deadline.Add(offset)。**

>maybeAddDeadline()

```go
// maybeAddDeadline potentially adds a deadline for the appropriate expected
// response for the passed wire protocol command to the pending responses map.
func (p *Peer) maybeAddDeadline(pendingResponses map[string]time.Time, msgCmd string) {
    // Setup a deadline for each message being sent that expects a response.
    //
    // NOTE: Pings are intentionally ignored here since they are typically
    // sent asynchronously and as a result of a long backlock of messages,
    // such as is typical in the case of initial block download, the
    // response won't be received in time.
    deadline := time.Now().Add(stallResponseTimeout)
    switch msgCmd {
    case wire.CmdVersion:
        // Expects a verack message.
        pendingResponses[wire.CmdVerAck] = deadline

    case wire.CmdMemPool:
        // Expects an inv message.
        pendingResponses[wire.CmdInv] = deadline

    case wire.CmdGetBlocks:
        // Expects an inv message.
        pendingResponses[wire.CmdInv] = deadline

    case wire.CmdGetData:
        // Expects a block, merkleblock, tx, or notfound message.
        pendingResponses[wire.CmdBlock] = deadline
        pendingResponses[wire.CmdMerkleBlock] = deadline
        pendingResponses[wire.CmdTx] = deadline
        pendingResponses[wire.CmdNotFound] = deadline

    case wire.CmdGetHeaders:
        // Expects a headers message.  Use a longer deadline since it
        // can take a while for the remote peer to load all of the
        // headers.
        deadline = time.Now().Add(stallResponseTimeout * 3)
        pendingResponses[wire.CmdHeaders] = deadline
    }
}
```

