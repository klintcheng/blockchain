
# 1. 节点peer
<!-- TOC -->

- [1. 节点peer](#1-节点peer)
    - [1.1. peer overview](#11-peer-overview)
    - [1.2. peer.start](#12-peerstart)
    - [1.3. peer握手](#13-peer握手)
        - [1.3.1. 发送version消息](#131-发送version消息)
        - [1.3.2. 读version消息](#132-读version消息)
        - [1.3.3. 应答ver消息](#133-应答ver消息)

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
```
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

```
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
> - stallHandler： 消息回复延时处理
> 
>   handles stall detection for the peer.  This entails keeping track of expected responses and assigning them deadlines while accounting for the time spent in callbacks.  It must be run as a goroutine.
> - inHandler: 处理进入的消息
> 
>   handles all incoming messages for the peer.  It must be run as a goroutine.
> - queueHandler： 处理节点出去的消息，写到sendQueue队列
> 
>   handles the queuing of outgoing data for the peer. This runs as a muxer for various sources of input so we can ensure that server and peer handlers will not block on us sending a message.  That data is then passed on to outHandler to be actually written.
> - outHandler： 处理sendQueue消息，发送出去
> 
>   handles all outgoing messages for the peer.  It must be run as a goroutine.  It uses a buffered channel to serialize output messages while allowing the sender to continue running asynchronously
> - pingHandler：心跳处理
> 
>   periodically pings the peer.  It must be run as a goroutine.

## 1.3. peer握手

当一个节点连接之后，它会发送自己的版本消息给对方，对方收到消息之后，也会回复自己的信息。

!!!* Once one or more connections are established, the new node will send an addr message containing its own IP address to its neighbors. The neighbors will, in turn, forward the addr message to their neighbors, ensuring that the newly connected node becomes well known and better connected. Additionally, the newly connected node can send getaddr to the neighbors, asking them to return a list of IP addresses of other peers. That way, a node can find peers to connect to and advertise its existence on the network for other nodes to find it. Address propagation and discovery shows the address discovery protocol. [mastering bitcoin]

在peer.start()中，首先会启用一个goroutine去处理握手。

```
negotiateErr := make(chan error, 1)
go func() {
    if p.inbound {
        negotiateErr <- p.negotiateInboundProtocol()
    } else {
        negotiateErr <- p.negotiateOutboundProtocol()
    }
}()
```
如果这个节点B是当前节点A接连过去的，那么它会调用negotiateOutboundProtocol。negotiateOutboundProtocol与negotiateInboundProtocol是一个必定是个相反的过程。

```
func (p *Peer) negotiateOutboundProtocol() error {
    if err := p.writeLocalVersionMsg(); err != nil {
        return err
    }

    return p.readRemoteVersionMsg()
}
```

```
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

可以看到，写消息是一个独立的逻辑，不依赖queueHandler等。创建一个消息之后，直接发送出去。

```
// writeLocalVersionMsg writes our version message to the remote peer.
func (p *Peer) writeLocalVersionMsg() error {
    localVerMsg, err := p.localVersionMsg()
    if err != nil {
        return err
    }

    return p.writeMessage(localVerMsg, wire.LatestEncoding)
}
```
这里创建MsgVersion是一个重点。涉及到两个节点是否能正常沟通。

> 创建MsgVersion

```
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
localVersionMsg方法就是把自己节点的相关信息封装到MsgVersion中。

```
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
```
// newestBlock returns the current best block hash and height using the format
// required by the configuration for the peer package.
func (sp *serverPeer) newestBlock() (*chainhash.Hash, int32, error) {
	best := sp.server.chain.BestSnapshot()
	return &best.Hash, best.Height, nil
}
```

### 1.3.2. 读version消息

读版本消息也是一个独立的逻辑。

```
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




```
// MsgVerAck defines a bitcoin verack message which is used for a peer to
// acknowledge a version message (MsgVersion) after it has used the information
// to negotiate parameters.  It implements the Message interface.
//
// This message has no payload.
type MsgVerAck struct{}

// Command returns the protocol command string for the message.  This is part
// of the Message interface implementation.
func (msg *MsgVerAck) Command() string {
    return CmdVerAck
}
```
其中CmdVerAck = "verack"，为常量。

>step.2 写入outputQueue

```
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

> step.3 缓冲队列

最终会创建一个outMsg放到outputQueue中。而outputQueue是在**queueHandler**中处理的。

```
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

通过调用queuePacket，消息最后还是推到sendQueue。这里封装了一下，当waiting=false 时，直接发送，否则加才到pendingMsgs中，收到发送完成的通知之后，再去pendingMsgs取一下个消息发送。

>step.4 消息发送处理
```
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

> 这个里，做了几件事：
>1. 发送一个sccSendMessage通知给stallHandler
>2. 发送消息到socket
>3. lastSend重置
>4. 写doneChan，唤醒等待的goroutine
>5. 写p.sendDoneQueue，通知queueHandler处理缓冲队列中其它消息

>step.5 消息发送到网络
```
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
- wire.WriteMessageWithEncodingN： 真正写消息到socket的方法。
- server.OnWrite： 用于通知server发送的数据量:
```
// OnWrite is invoked when a peer sends a message and it is used to update
// the bytes sent by the server.
func (sp *serverPeer) OnWrite(_ *peer.Peer, bytesWritten int, msg wire.Message, err error) {
	sp.server.AddBytesSent(uint64(bytesWritten))
}
```

### 1.3.3. 应答ver消息

当一个节点A连结到节点B，并发送了VerAck时，B节点收到消息要处理。回到inHandler()，这个处理器是用于接收消息并处理。

```
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

        ... MsgAddr/MsgGetAddr/MsgAddr/MsgPing/MsgPong/ more..

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