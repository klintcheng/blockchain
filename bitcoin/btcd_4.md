# 1. 节点连结
<!-- TOC -->

- [1. 节点连结](#1-节点连结)
    - [1.1. 启动连接管理](#11-启动连接管理)
        - [1.1.1. 启动监听处理](#111-启动监听处理)
            - [1.1.1.1. net.Listen](#1111-netlisten)
            - [1.1.1.2. listener.Accept()](#1112-listeneraccept)
    - [1.2. 创建第一个连接](#12-创建第一个连接)

<!-- /TOC -->
## 1.1. 启动连接管理

上一章讲了节点地址服务。当前节点得到一些可用地址之后，就会建立连接，维护起来。在这个章，我们来看看第一个连结是如何建立的。首先，我们从go s.connManager.Start()开始，看看这个服务启动时做了什么事。

```
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

Start 做了三件事：
- 启动连接处理器
- 启动端口监听处理
- 创建一些接连（默认8个）

### 1.1.1. 启动监听处理

在[server.NewServer](btcd_2.md#13-newserver)中，会调用initListeners启用监听器。然后这些监听器作为配置项传入connmanager。然后在Start时开始处理节点请求。先来看看网络监听器的创建：

#### 1.1.1.1. net.Listen

```
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

#### 1.1.1.2. listener.Accept()

```
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

** 这个回调方法就会创建一个ServerPeer：**

- 这里创建了一个新的节点
- 判断是否在白名单中。
- 创建一个InboundPeer
- 关连conn到serverPeer中
- 等待节点退出通知


## 1.2. 创建第一个连接

上面的逻辑是被动建立一个连接。下面我们来看看，得到种子节点之后，是如何主动去连接一个节点的。

在connManager.Start()中会默认主动连接一些节点：
```
for i := atomic.LoadUint64(&cm.connReqCount); i < uint64(cm.cfg.TargetOutbound); i++ {
    go cm.NewConnReq()
}

<!-- connReqCount: 初始为0 -->
<!-- TargetOutbound: 默认为8（如果配置文件中MaxPeers没有设置） -->
```

我们看下NewConnReq方法：

```
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

要建立连接，必须要得到一个节点地址，因此，这里cm.cfg.GetNewAddress必不可少。这个方法就是NewServer中创建的newAddressFunc。
然后创建一个registerPending写到无缓冲的requests通道中。




```
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