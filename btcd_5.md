# 1. 连接管理

<!-- TOC -->

- [1. 连接管理](#1-连接管理)
    - [1.1. 简介](#11-简介)
    - [1.2. 启动连接管理](#12-启动连接管理)
        - [1.2.1. 启动监听处理](#121-启动监听处理)
            - [1.2.1.1. net.Listen](#1211-netlisten)
            - [1.2.1.2. listener.Accept()](#1212-listeneraccept)
        - [1.2.2. 创建第一个连接](#122-创建第一个连接)
            - [1.2.2.1. NewConnReq](#1221-newconnreq)
            - [1.2.2.2. 实现接连Connect](#1222-实现接连connect)
        - [1.2.3. 连接失败](#123-连接失败)
            - [1.2.3.1. 失败情况](#1231-失败情况)
            - [1.2.3.2. 失败处理](#1232-失败处理)
    - [1.3. 断开连结](#13-断开连结)
        - [1.3.1. 节点断开监听](#131-节点断开监听)
        - [1.3.2. 处理节点断开](#132-处理节点断开)
        - [1.3.3. 处理连接断开](#133-处理连接断开)

<!-- /TOC -->

## 1.1. 简介

ConnManager负责管理当前节点所有的连接管理，并对所有连接监听处理。并且把创建好的连接返回给server。

## 1.2. 启动连接管理

上一章讲了节点地址服务。当前节点得到一些可用地址之后，就会建立连接，维护起来。在这个章，我们来看看第一个连结是如何建立的。首先，我们从go s.connManager.Start()开始，看看这个服务启动时做了什么事。

```go
// Start launches the connection manager and begins connecting to the network.
func (cm *ConnManager) Start() {
    // Already started?
    if atomic.AddInt32(&cm.start, 1) != 1 {
        return
    }

    log.Trace("Connection manager started")
    cm.wg.Add(1)
    go cm.connHandler()

    // Start all the listeners so long as the caller requested them and
    // provided a callback to be invoked when connections are accepted.
    if cm.cfg.OnAccept != nil {
        for _, listner := range cm.cfg.Listeners {
            cm.wg.Add(1)
            go cm.listenHandler(listner)
        }
    }

    for i := atomic.LoadUint64(&cm.connReqCount); i < uint64(cm.cfg.TargetOutbound); i++ {
        go cm.NewConnReq()
    }
}
```

>Start 做了三件事：

- 启动连接处理器
- 启动端口监听处理
- 创建一些接连（默认8个）

### 1.2.1. 启动监听处理

在[server.NewServer](btcd_2.md#13-newserver)中，会调用initListeners启用监听器。然后这些监听器作为配置项传入connmanager。然后在Start时开始处理节点请求。先来看看网络监听器的创建：

#### 1.2.1.1. net.Listen

```go
// initListeners initializes the configured net listeners and adds any bound
// addresses to the address manager. Returns the listeners and a NAT interface,
// which is non-nil if UPnP is in use.
func initListeners(amgr *addrmgr.AddrManager, listenAddrs []string, services wire.ServiceFlag) ([]net.Listener, NAT, error) {
    // Listen for TCP connections at the configured addresses
    netAddrs, err := parseListeners(listenAddrs)
    if err != nil {
        return nil, nil, err
    }

    listeners := make([]net.Listener, 0, len(netAddrs))
    for _, addr := range netAddrs {
        listener, err := net.Listen(addr.Network(), addr.String())
        if err != nil {
            srvrLog.Warnf("Can't listen on %s: %v", addr, err)
            continue
        }
        listeners = append(listeners, listener)
    }
    ...

    return listeners, nat, nil
}
```

由于我们没有配置，默认情况会启用ip4和ip6的tcp端口监听。

#### 1.2.1.2. listener.Accept()

```go
// listenHandler accepts incoming connections on a given listener.  It must be
// run as a goroutine.
func (cm *ConnManager) listenHandler(listener net.Listener) {
    log.Infof("Server listening on %s", listener.Addr())
    for atomic.LoadInt32(&cm.stop) == 0 {
        conn, err := listener.Accept()
        if err != nil {
            // Only log the error if not forcibly shutting down.
            if atomic.LoadInt32(&cm.stop) == 0 {
                log.Errorf("Can't accept connection: %v", err)
            }
            continue
        }
        go cm.cfg.OnAccept(conn)
    }

    cm.wg.Done()
    log.Tracef("Listener handler done for %s", listener.Addr())
}
```
在这个goroutine中会一直循环等待，当有新的连接进入时，就会马上创建一个新的goroutine去处理（和net.httpserver里的逻辑一样）。
cm.cfg.OnAccept方法就是传入的server.inboundPeerConnected(见NewServer)。我们进去看看。

```
// inboundPeerConnected is invoked by the connection manager when a new inbound
// connection is established.  It initializes a new inbound server peer
// instance, associates it with the connection, and starts a goroutine to wait
// for disconnection.
func (s *server) inboundPeerConnected(conn net.Conn) {
    sp := newServerPeer(s, false)
    sp.isWhitelisted = isWhitelisted(conn.RemoteAddr())
    sp.Peer = peer.NewInboundPeer(newPeerConfig(sp))
    sp.AssociateConnection(conn)
    go s.peerDoneHandler(sp)
}
```

>**这个回调方法就会创建一个ServerPeer, 分为如下几步**

- 这里创建了一个新的节点
- 判断是否在白名单中。
- 创建一个InboundPeer
- 关连conn到serverPeer中
- 等待节点退出通知


### 1.2.2. 创建第一个连接

上面的逻辑是被动建立一个连接。下面我们来看看，得到种子节点之后，是如何主动去连接一个节点的。

>在connManager.Start()中会默认主动连接一些节点：

```go
for i := atomic.LoadUint64(&cm.connReqCount); i < uint64(cm.cfg.TargetOutbound); i++ {
    go cm.NewConnReq()
}

<!-- connReqCount: 初始为0 -->
<!-- TargetOutbound: 默认为8（如果配置文件中MaxPeers没有设置） -->
```

#### 1.2.2.1. NewConnReq

```go
// NewConnReq creates a new connection request and connects to the
// corresponding address.
func (cm *ConnManager) NewConnReq() {
    if atomic.LoadInt32(&cm.stop) != 0 {
        return
    }
    if cm.cfg.GetNewAddress == nil {
        return
    }

    c := &ConnReq{}
    atomic.StoreUint64(&c.id, atomic.AddUint64(&cm.connReqCount, 1))

    // Submit a request of a pending connection attempt to the connection
    // manager. By registering the id before the connection is even
    // established, we'll be able to later cancel the connection via the
    // Remove method.
    done := make(chan struct{})
    select {
    case cm.requests <- registerPending{c, done}:
    case <-cm.quit:
        return
    }

    // Wait for the registration to successfully add the pending conn req to
    // the conn manager's internal state.
    select {
    case <-done:
    case <-cm.quit:
        return
    }

    addr, err := cm.cfg.GetNewAddress()
    if err != nil {
        select {
        case cm.requests <- handleFailed{c, err}:
        case <-cm.quit:
        }
        return
    }

    c.Addr = addr

    cm.Connect(c)
}
```

- 创建ConnReq并设置id
- 创建一个registerPending写到无缓冲的requests通道中
- 等待处理之后的通知(channal done)
- 调用GetNewAddress得到地址（这个方法就是NewServer中创建的newAddressFunc）
- cm.Connect(c)
  
我们看下在connHandler的处理：

```go
select {
case req := <-cm.requests:
    switch msg := req.(type) {

    case registerPending:
        connReq := msg.c
        connReq.updateState(ConnPending)
        pending[msg.c.id] = connReq
        close(msg.done)
     
```

这个处理很简单，更新状态，添加到pending中，close(msg.done)之后会唤醒NewConnReq

>newAddressFunc 方法就是调用 s.addrManager.GetAddress()得到地址。

```go
newAddressFunc = func() (net.Addr, error) {
    for tries := 0; tries < 100; tries++ {
        addr := s.addrManager.GetAddress()
        if addr == nil {
            break
        }

        // Address will not be invalid, local or unroutable
        // because addrmanager rejects those on addition.
        // Just check that we don't already have an address
        // in the same group so that we are not connecting
        // to the same network segment at the expense of
        // others.
        key := addrmgr.GroupKey(addr.NetAddress())
        if s.OutboundGroupCount(key) != 0 {
            continue
        }

        // only allow recent nodes (10mins) after we failed 30
        // times
        if tries < 30 && time.Since(addr.LastAttempt()) < 10*time.Minute {
            continue
        }

        // allow nondefault ports after 50 failed tries.
        if tries < 50 && fmt.Sprintf("%d", addr.NetAddress().Port) !=
            activeNetParams.DefaultPort {
            continue
        }

        addrString := addrmgr.NetAddressKey(addr.NetAddress())
        return addrStringToNetAddr(addrString)
    }

    return nil, errors.New("no valid connect address")
}
```

#### 1.2.2.2. 实现接连Connect

开始拨号连接，cm.cfg.Dial(c.Addr)调用的方法就是net.Dial。连接成功之后，就通知处理器处理(connHandler)。

```go
// Connect assigns an id and dials a connection to the address of the
// connection request.
func (cm *ConnManager) Connect(c *ConnReq) {
    if atomic.LoadInt32(&cm.stop) != 0 {
        return
    }
    if atomic.LoadUint64(&c.id) == 0 {
        ...
    }

    log.Debugf("Attempting to connect to %v", c)

    conn, err := cm.cfg.Dial(c.Addr)
    if err != nil {
        select {
        case cm.requests <- handleFailed{c, err}:
        case <-cm.quit:
        }
        return
    }

    select {
    case cm.requests <- handleConnected{c, conn}:
    case <-cm.quit:
    }
}
```

**connHandler**:

```go
case handleConnected:
    connReq := msg.c

    if _, ok := pending[connReq.id]; !ok {
        if msg.conn != nil {
            msg.conn.Close()
        }
        log.Debugf("Ignoring connection for "+
            "canceled connreq=%v", connReq)
        continue
    }

    connReq.updateState(ConnEstablished)
    connReq.conn = msg.conn
    conns[connReq.id] = connReq
    log.Debugf("Connected to %v", connReq)
    connReq.retryCount = 0
    cm.failedAttempts = 0

    delete(pending, connReq.id)

    if cm.cfg.OnConnection != nil {
        go cm.cfg.OnConnection(connReq, msg.conn)
    }
```

上面的处理了完成之后，又会起一个新的goroutine。调用server.outboundPeerConnected.
```
cmgr, err := connmgr.New(&connmgr.Config{
    Listeners:      listeners,
    OnAccept:       s.inboundPeerConnected,
    RetryDuration:  connectionRetryInterval,
    TargetOutbound: uint32(targetOutbound),
    Dial:           btcdDial,
    OnConnection:   s.outboundPeerConnected,
    GetNewAddress:  newAddressFunc,
})
```

>**出去的连接，与inboundPeerConnected方法类似，会新建立一个ServerPeer**

```go
// outboundPeerConnected is invoked by the connection manager when a new
// outbound connection is established.  It initializes a new outbound server
// peer instance, associates it with the relevant state such as the connection
// request instance and the connection itself, and finally notifies the address
// manager of the attempt.
func (s *server) outboundPeerConnected(c *connmgr.ConnReq, conn net.Conn) {
    sp := newServerPeer(s, c.Permanent)
    p, err := peer.NewOutboundPeer(newPeerConfig(sp), c.Addr.String())
    if err != nil {
        srvrLog.Debugf("Cannot create outbound peer %s: %v", c.Addr, err)
        s.connManager.Disconnect(c.ID())
    }
    sp.Peer = p
    sp.connReq = c
    sp.isWhitelisted = isWhitelisted(conn.RemoteAddr())
    sp.AssociateConnection(conn)
    go s.peerDoneHandler(sp)
    s.addrManager.Attempt(sp.NA())
}
```
同时，我们看到的地址尝试次数的修改s.addrManager.Attempt(sp.NA())：
```
// Attempt increases the given address' attempt counter and updates
// the last attempt time.
func (a *AddrManager) Attempt(addr *wire.NetAddress) {
    a.mtx.Lock()
    defer a.mtx.Unlock()

    // find address.
    // Surely address will be in tried by now?
    ka := a.find(addr)
    if ka == nil {
        return
    }
    // set last tried time to now
    ka.attempts++
    ka.lastattempt = time.Now()
}
```

这个方法中，会修改KnownAddress中的 **attempts 和 lastattempt**。
目前为止，地址相关的几个属性都用到了。还有一个lastsuccess没有出现，这个属性是在Good方法中做的修改，先不管它是在什么情况下调用。

```go
// Good marks the given address as good.  To be called after a successful
// connection and version exchange.  If the address is unknown to the address
// manager it will be ignored.
func (a *AddrManager) Good(addr *wire.NetAddress) {
    ...
    // ka.Timestamp is not updated here to avoid leaking information
    // about currently connected peers.
    now := time.Now()
    ka.lastsuccess = now
    ka.lastattempt = now
    ka.attempts = 0
    ...
}
```

至此，连进来的节点逻辑和连接出去的节点逻辑大致分析完。我们来看看连接失败情况。

### 1.2.3. 连接失败

>两种情况下会失败，会发一个失败请求：

#### 1.2.3.1. 失败情况

- GetNewAddress error
  
```go
addr, err := cm.cfg.GetNewAddress()
if err != nil {
    select {
    case cm.requests <- handleFailed{c, err}:
    case <-cm.quit:
    }
    return
}
```

- Dial error

```go
conn, err := cm.cfg.Dial(c.Addr)
if err != nil {
    select {
    case cm.requests <- handleFailed{c, err}:
    case <-cm.quit:
    }
    return
}
```

#### 1.2.3.2. 失败处理

```go
func (cm *ConnManager) connHandler() {
...
case handleFailed:
    connReq := msg.c

    if _, ok := pending[connReq.id]; !ok {
        log.Debugf("Ignoring connection for "+
            "canceled conn req: %v", connReq)
        continue
    }

    connReq.updateState(ConnFailing)
    log.Debugf("Failed to connect to %v: %v",
        connReq, msg.err)
    cm.handleFailedConn(connReq)
}
```

```go
// handleFailedConn handles a connection failed due to a disconnect or any
// other failure. If permanent, it retries the connection after the configured
// retry duration. Otherwise, if required, it makes a new connection request.
// After maxFailedConnectionAttempts new connections will be retried after the
// configured retry duration.
func (cm *ConnManager) handleFailedConn(c *ConnReq) {
    if atomic.LoadInt32(&cm.stop) != 0 {
        return
    }
    if c.Permanent {
        c.retryCount++
        d := time.Duration(c.retryCount) * cm.cfg.RetryDuration
        if d > maxRetryDuration {
            d = maxRetryDuration
        }
        log.Debugf("Retrying connection to %v in %v", c, d)
        time.AfterFunc(d, func() {
            cm.Connect(c)
        })
    } else if cm.cfg.GetNewAddress != nil {
        cm.failedAttempts++
        if cm.failedAttempts >= maxFailedAttempts {
            log.Debugf("Max failed connection attempts reached: [%d] "+
                "-- retrying connection in: %v", maxFailedAttempts,
                cm.cfg.RetryDuration)
            time.AfterFunc(cm.cfg.RetryDuration, func() {
                cm.NewConnReq()
            })
        } else {
            go cm.NewConnReq()
        }
    }
}
```

>失败情况下，会有两种处理方式：

1. 永久连接，在一定时间之后直接重连。
2. 非永久连接，调用NewConnReq，获取另一个地址去连接。

Permanent默认为false。 在server.NewServer()时，如果有配置固定节点，会当作永久接连。

```go
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

## 1.3. 断开连结

正常情况下，收到退出事情（channal p.quit）时，会断开连结。我们来来退出的流程处理：

### 1.3.1. 节点断开监听

在节点创建时，会监听退出事件。

```go
func (s *server) outboundPeerConnected(c *connmgr.ConnReq, conn net.Conn) {
    sp := newServerPeer(s, c.Permanent)
    ...
    go s.peerDoneHandler(sp)
    ...
}

func (s *server) peerDoneHandler(sp *serverPeer) {
    sp.WaitForDisconnect()
    s.donePeers <- sp
    ...
}

func (p *Peer) WaitForDisconnect() {
    <-p.quit
}
```

>节点处理器：

```go
func (s *server) peerHandler() {
    ...
    // Disconnected peers.
    case p := <-s.donePeers:
        s.handleDonePeerMsg(state, p)
    ...
```

### 1.3.2. 处理节点断开

```go
// handleDonePeerMsg deals with peers that have signalled they are done.  It is
// invoked from the peerHandler goroutine.
func (s *server) handleDonePeerMsg(state *peerState, sp *serverPeer) {
    ...

    if sp.connReq != nil {
        s.connManager.Disconnect(sp.connReq.ID())
    }

    // Update the address' last seen time if the peer has acknowledged
    // our version and has sent us its version as well.
    if sp.VerAckReceived() && sp.VersionKnown() && sp.NA() != nil {
        s.addrManager.Connected(sp.NA())
    }

    // If we get here it means that either we didn't know about the peer
    // or we purposefully deleted it.
}
```

- 处理连接断开
- 处理地址  
    ```修改当前连接的NetAddress中的Timestamp（Last time the address was seen.）```

### 1.3.3. 处理连接断开

调用Disconnect发送断开通知到连接处理器goroutine中处理：

```go
// Disconnect disconnects the connection corresponding to the given connection
// id. If permanent, the connection will be retried with an increasing backoff
// duration.
func (cm *ConnManager) Disconnect(id uint64) {
    if atomic.LoadInt32(&cm.stop) != 0 {
        return
    }

    select {
    case cm.requests <- handleDisconnected{id, true}:
    case <-cm.quit:
    }
}
```

开始处理连接断开：

```go
case handleDisconnected:
    connReq, ok := conns[msg.id]
    if !ok {
        connReq, ok = pending[msg.id]
        if !ok {
            log.Errorf("Unknown connid=%d",
                msg.id)
            continue
        }

        // Pending connection was found, remove
        // it from pending map if we should
        // ignore a later, successful
        // connection.
        connReq.updateState(ConnCanceled)
        log.Debugf("Canceling: %v", connReq)
        delete(pending, msg.id)
        continue

    }

    // An existing connection was located, mark as
    // disconnected and execute disconnection
    // callback.
    log.Debugf("Disconnected from %v", connReq)
    delete(conns, msg.id)

    if connReq.conn != nil {
        connReq.conn.Close()
    }

    if cm.cfg.OnDisconnection != nil {
        go cm.cfg.OnDisconnection(connReq)
    }

    // All internal state has been cleaned up, if
    // this connection is being removed, we will
    // make no further attempts with this request.
    if !msg.retry {
        connReq.updateState(ConnDisconnected)
        continue
    }

    // Otherwise, we will attempt a reconnection if
    // we do not have enough peers, or if this is a
    // persistent peer. The connection request is
    // re added to the pending map, so that
    // subsequent processing of connections and
    // failures do not ignore the request.
    if uint32(len(conns)) < cm.cfg.TargetOutbound ||
        connReq.Permanent {

        connReq.updateState(ConnPending)
        log.Debugf("Reconnecting to %v",
            connReq)
        pending[msg.id] = connReq
        cm.handleFailedConn(connReq)
}
```

- 清理工作
- 如果传入的retry为true，直接结束(主动断开情况下，retry=true)
- 否则，尝试重新连接（满足条件）。

