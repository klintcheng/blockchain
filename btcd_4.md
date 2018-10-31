
# 1. 节点发现与维护

<!-- TOC -->

- [1. 节点发现与维护](#1-节点发现与维护)
    - [简介](#简介)
    - [1.1. 启动AddrManager](#11-启动addrmanager)
        - [1.1.1. 本地读取地址信息](#111-本地读取地址信息)
        - [1.1.2. 同步到本地文件](#112-同步到本地文件)
        - [1.1.3. 从种子源获取地址](#113-从种子源获取地址)
    - [1.2. 获取随机地址方法](#12-获取随机地址方法)
        - [1.2.1. GetAddress](#121-getaddress)
    - [1.3. 从连接的节点获取](#13-从连接的节点获取)
        - [1.3.1. 获取节点地址](#131-获取节点地址)
        - [1.3.2. 把地址给其它节点](#132-把地址给其它节点)

<!-- /TOC -->

## 1.1. 简介

AddrManager作为网络通信最基础的管理器，主要负责维护可用节点，记录节点的连接等信息。提供上层获取其它节点的网络通信地址。

## 1.2. 启动AddrManager

一个节点在启动时，必须连结到一些可用的节点上，才能共同区块数。保持全网络数据一致性。

在第二章中，我们有看到创建地址管理服务时，其中一个参数就是peers.json。但是，第一次启动节点时，节点是没有任何可用的临近节点的，那这个节点是如何连结到主网的呢。下面就从代码中找找逻辑：

首先，还是回到peerhandler中，看看addrmgr.start()做了什么事。

```go
// Start begins the core address handler which manages a pool of known
// addresses, timeouts, and interval based writes.
func (a *AddrManager) Start() {
    // Already started?
    if atomic.AddInt32(&a.started, 1) != 1 {
        return
    }

    log.Trace("Starting address manager")

    // Load peers we already know about from file.
    a.loadPeers()

    // Start the address ticker to save addresses periodically.
    a.wg.Add(1)
    go a.addressHandler()
}
```

### 1.2.1. 本地读取地址信息

Start加载地址信息的代码：a.loadPeers()。它是从文件中加载记录的节点地址信息。

```go
// loadPeers loads the known address from the saved file.  If empty, missing, or
// malformed file, just don't load anything and start fresh
func (a *AddrManager) loadPeers() {
    a.mtx.Lock()
    defer a.mtx.Unlock()

    err := a.deserializePeers(a.peersFile)
    if err != nil {
        log.Errorf("Failed to parse file %s: %v", a.peersFile, err)
        // if it is invalid we nuke the old one unconditionally.
        err = os.Remove(a.peersFile)
        if err != nil {
            log.Warnf("Failed to remove corrupt peers file %s: %v",
                a.peersFile, err)
        }
        a.reset()
        return
    }
    log.Infof("Loaded %d addresses from file '%s'", a.numAddresses(), a.peersFile)
}
```

读取文件会用到锁。然后调用a.deserializePeers(a.peersFile)读取文件，文件内容是json格式，因此，会反序列化为对象serializedAddrManager。我们来看下相关内容：

**peers.json内容格式**

```go
{
    "Version": 1,
    "Key": [
        249,
        ...
        232,
        239
    ],
    "Addresses": [
        {
            "Addr": "80.219.27.212:8333",
            "Src": "13.115.194.245:8333",
            "Attempts": 0,
            "TimeStamp": 1540282028,
            "LastAttempt": -62135596800,
            "LastSuccess": -62135596800
        },
        {
            "Addr": "2.92.255.216:8333",
            "Src": "13.115.194.245:8333",
            "Attempts": 0,
            "TimeStamp": 1540282028,
            "LastAttempt": -62135596800,
            "LastSuccess": -62135596800
        }
    ],
    "NewBuckets": [
        [],
        [],
        [],
        [
            "[2001:0:9d38:6ab8:f8:10c3:fad3:562d]:8333"
        ],
        [
            "18.194.141.225:8333",
            "[2605:6000:101c:6cc:7062:2ca7:9174:731c]:8333",
            "136.56.155.94:8333",
            "49.81.66.104:8333",
            "95.114.56.217:8333",
            "49.81.65.19:8333",
            "[2605:6000:1105:111f:dc86:e4e6:c82e:7556]:8333",
            "77.37.102.6:8333",
            "1.65.182.72:8333",
            "49.81.66.69:8333",
            "18.144.39.12:8333",
            "114.206.60.98:8333"
        ]
        ...省略
    ]
    "TriedBuckets": [
        [],
        [],
        [],
        [],
        [
            "74.14.27.89:8333"
        ],
        [
            "13.115.194.245:8333"
        ]
        ...省略
    ]
}
```

**对应的结构体如下**

```
type serializedAddrManager struct {
    Version      int
    Key          [32]byte
    Addresses    []*serializedKnownAddress
    NewBuckets   [newBucketCount][]string // string is NetAddressKey
    TriedBuckets [triedBucketCount][]string
}

type serializedKnownAddress struct {
    Addr        string
    Src         string
    Attempts    int
    TimeStamp   int64
    LastAttempt int64
    LastSuccess int64
    // no refcount or tried, that is available from context.
}

// KnownAddress tracks information about a known network address that is used
// to determine how viable an address is.
type KnownAddress struct {
    na          *wire.NetAddress
    srcAddr     *wire.NetAddress
    attempts    int
    lastattempt time.Time
    lastsuccess time.Time
    tried       bool
    refs        int // reference count of new buckets
}

```

这里先不管它们的意思。大致知道节点地址配置就行了。后面一步步看。

读取的KnownAddress最终会保存到addrIndex中：

```go
func (a *AddrManager) deserializePeers(filePath string) error {

    _, err := os.Stat(filePath)
    if os.IsNotExist(err) {
        return nil
    }
    r, err := os.Open(filePath)
    if err != nil {
        return fmt.Errorf("%s error opening file: %v", filePath, err)
    }
    defer r.Close()

    var sam serializedAddrManager
    dec := json.NewDecoder(r)
    err = dec.Decode(&sam)
    if err != nil {
        return fmt.Errorf("error reading %s: %v", filePath, err)
    }

    if sam.Version != serialisationVersion {
        return fmt.Errorf("unknown version %v in serialized "+
            "addrmanager", sam.Version)
    }
    copy(a.key[:], sam.Key[:])

    for _, v := range sam.Addresses {
        ka := new(KnownAddress)
        ka.na, err = a.DeserializeNetAddress(v.Addr)
        if err != nil {
            return fmt.Errorf("failed to deserialize netaddress "+
                "%s: %v", v.Addr, err)
        }
        ka.srcAddr, err = a.DeserializeNetAddress(v.Src)
        if err != nil {
            return fmt.Errorf("failed to deserialize netaddress "+
                "%s: %v", v.Src, err)
        }
        ka.attempts = v.Attempts
        ka.lastattempt = time.Unix(v.LastAttempt, 0)
        ka.lastsuccess = time.Unix(v.LastSuccess, 0)
        a.addrIndex[NetAddressKey(ka.na)] = ka
    }
    ...
}
```

> NetAddressKey返回的格式为：
>- IPv4  
>  ip:port
>- IPv6  
>  [ip]:port

### 1.2.2. 同步到本地文件

同时，启动一个goroutine去定时处理地址。

```go
// addressHandler is the main handler for the address manager.  It must be run
// as a goroutine.
func (a *AddrManager) addressHandler() {
    dumpAddressTicker := time.NewTicker(dumpAddressInterval)
    defer dumpAddressTicker.Stop()
out:
    for {
        select {
        case <-dumpAddressTicker.C:
            a.savePeers()

        case <-a.quit:
            break out
        }
    }
    a.savePeers()
    a.wg.Done()
    log.Trace("Address handler done")
}
```

这里，代码比较简单。间隔10分钟保存一次内存中的节点数据到配置文件peers.json中:
a.savePeers()与a.loadPeers()是一个相反的过程。

到目前为止，peers.json是没有内容的，因此 ，我们还要找到读取种子的代码。

### 1.2.3. 从种子源获取地址

回到server.start()中，在s.addrManager.Start()启动之后看到了相关代码。**

```go
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

```

第一个参数就是在chaincfg包中硬编码的。第二个参数是常量。

```go
// MainNetParams defines the network parameters for the main Bitcoin network.
var MainNetParams = Params{
    Name:        "mainnet",
    Net:         wire.MainNet,
    DefaultPort: "8333",
    DNSSeeds: []DNSSeed{
        {"seed.bitcoin.sipa.be", true},
        {"dnsseed.bluematt.me", true},
        {"dnsseed.bitcoin.dashjr.org", false},
        {"seed.bitcoinstats.com", true},
        {"seed.bitnodes.io", false},
        {"seed.bitcoin.jonasschnelli.ch", true},
    },
    ...
}
```

第三个参数是个方法。这个方法就是解析dns.传入dns,返回一组ip。

```go
// btcdLookup resolves the IP of the given host using the correct DNS lookup
// function depending on the configuration options.  For example, addresses will
// be resolved using tor when the --proxy flag was specified unless --noonion
// was also specified in which case the normal system DNS resolver will be used.
//
// Any attempt to resolve a tor address (.onion) will return an error since they
// are not intended to be resolved outside of the tor proxy.
func btcdLookup(host string) ([]net.IP, error) {
    if strings.HasSuffix(host, ".onion") {
        return nil, fmt.Errorf("attempt to resolve tor address %s", host)
    }

    return cfg.lookup(host)
}
```

我们进入到这个方法内看看：

```go
// SeedFromDNS uses DNS seeding to populate the address manager with peers.
func SeedFromDNS(chainParams *chaincfg.Params, reqServices wire.ServiceFlag,
    lookupFn LookupFunc, seedFn OnSeed) {

    for _, dnsseed := range chainParams.DNSSeeds {
        var host string
        if !dnsseed.HasFiltering || reqServices == wire.SFNodeNetwork {
            host = dnsseed.Host
        } else {
            host = fmt.Sprintf("x%x.%s", uint64(reqServices), dnsseed.Host)
        }

        go func(host string) {
            randSource := mrand.New(mrand.NewSource(time.Now().UnixNano()))

            seedpeers, err := lookupFn(host)
            if err != nil {
                log.Infof("DNS discovery failed on seed %s: %v", host, err)
                return
            }
            numPeers := len(seedpeers)

            log.Infof("%d addresses found from DNS seed %s", numPeers, host)

            if numPeers == 0 {
                return
            }
            addresses := make([]*wire.NetAddress, len(seedpeers))
            // if this errors then we have *real* problems
            intPort, _ := strconv.Atoi(chainParams.DefaultPort)
            for i, peer := range seedpeers {
                addresses[i] = wire.NewNetAddressTimestamp(
                    // bitcoind seeds with addresses from
                    // a time randomly selected between 3
                    // and 7 days ago.
                    time.Now().Add(-1*time.Second*time.Duration(secondsIn3Days+
                        randSource.Int31n(secondsIn4Days))),
                    0, peer, uint16(intPort))
            }

            seedFn(addresses)
        }(host)
    }
}
```

由于解析nds是一次网络io操作，因此这里会都放到一个goroutine里处理。这也是为什么最后一个参数是个回调函数的原因了。

seedpeers, err := lookupFn(host) 会返回一组节点ip。最后包装成wire.NetAddress对象。添加到地址管理器中，我们来看下这个方法内部：

**s.addrManager.AddAddresses(addrs, addrs[0])**

```go
// AddAddresses adds new addresses to the address manager.  It enforces a max
// number of addresses and silently ignores duplicate addresses.  It is
// safe for concurrent access.
func (a *AddrManager) AddAddresses(addrs []*wire.NetAddress, srcAddr *wire.NetAddress) {
    a.mtx.Lock()
    defer a.mtx.Unlock()

    for _, na := range addrs {
        a.updateAddress(na, srcAddr)
    }
}
```
可以看到每个dns解析出的ip列表中，第一个就是源，此时，我们就知道为什么前面的KnownAddress中为什么会有两个地址了，一个na,一个srcAddr, 至于有什么用，先不管。

这里调用updateAddress。如果地址已经存在就会更新，否则就添加一条。

```go
// updateAddress is a helper function to either update an address already known
// to the address manager, or to add the address if not already known.
func (a *AddrManager) updateAddress(netAddr, srcAddr *wire.NetAddress) {
    // Filter out non-routable addresses. Note that non-routable
    // also includes invalid and local addresses.
    if !IsRoutable(netAddr) {
        return
    }

    addr := NetAddressKey(netAddr)
    ka := a.find(netAddr)
    if ka != nil {
       ...
    } else {
        // Make a copy of the net address to avoid races since it is
        // updated elsewhere in the addrmanager code and would otherwise
        // change the actual netaddress on the peer.
        netAddrCopy := *netAddr
        ka = &KnownAddress{na: &netAddrCopy, srcAddr: srcAddr}
        a.addrIndex[addr] = ka
        a.nNew++
        // XXX time penalty?
    }

    bucket := a.getNewBucket(netAddr, srcAddr)

    // Already exists?
    if _, ok := a.addrNew[bucket][addr]; ok {
        return
    }

    // Enforce max addresses.
    if len(a.addrNew[bucket]) > newBucketSize {
        log.Tracef("new bucket is full, expiring old")
        a.expireNew(bucket)
    }

    // Add to new bucket.
    ka.refs++
    a.addrNew[bucket][addr] = ka

    log.Tracef("Added new address %s for a total of %d addresses", addr,
        a.nTried+a.nNew)
}
```
这里看到有调用getNewBucket，得到一个bucket号，原理类似于hashmap。

**getNewBucket算法：**

```go
doublesha256(key + sourcegroup + int64(doublesha256(key + group + sourcegroup))%bucket_per_source_group) % num_new_buckets
```

系统设置一个bucket有64个地址。总共有1024个bucket。到这里我们就搞明白了前面**peers.json中NewBuckets为什么会有空的数组子元素了**

由于存储空间是不可变的，因此添加一个地址之后会去检查，如果操过size,就把旧的删除。

>**看下这个方法的说明：**

先删除坏的地址，如果没有，就选择一个最旧的地址删除。会用到na.TTimestamp比较

```go
// expireNew makes space in the new buckets by expiring the really bad entries.
// If no bad entries are available we look at a few and remove the oldest.
func (a *AddrManager) expireNew(bucket int) {
    // First see if there are any entries that are so bad we can just throw
    // them away. otherwise we throw away the oldest entry in the cache.
    // Bitcoind here chooses four random and just throws the oldest of
    // those away, but we keep track of oldest in the initial traversal and
    // use that information instead.
    var oldest *KnownAddress
    for k, v := range a.addrNew[bucket] {
        if v.isBad() {
            log.Tracef("expiring bad address %v", k)
            delete(a.addrNew[bucket], k)
            v.refs--
            if v.refs == 0 {
                a.nNew--
                delete(a.addrIndex, k)
            }
            continue
        }
        if oldest == nil {
            oldest = v
        } else if !v.na.Timestamp.After(oldest.na.Timestamp) {
            oldest = v
        }
    }
    ....
}
```

>**坏地址的判断：**

- It claims to be from the future
- It hasn't been seen in over a month
- It has failed at least three times and never succeeded
- It has failed ten times in the last week

>到此时，我们就可以理解KnownAddress中的三个属性是干什么用的了

>- attempts : 尝试次数
>- lastattempt : 最后一次尝试时间
>- lastsuccess : 最后一次成功时间

```go
// numRetries is the number of tried without a single success before
// we assume an address is bad.
const numRetries = 3

// numMissingDays is the number of days before which we assume an
// address has vanished if we have not seen it announced  in that long.
const numMissingDays = 30

// minBadDays is the number of days since the last success before we
// will consider evicting an address.
const minBadDays = 7

// maxFailures is the maximum number of failures we will accept without
// a success before considering an address bad.
const maxFailures = 10

func (ka *KnownAddress) isBad() bool {
    if ka.lastattempt.After(time.Now().Add(-1 * time.Minute)) {
        return false
    }

    // From the future?
    if ka.na.Timestamp.After(time.Now().Add(10 * time.Minute)) {
        return true
    }

    // Over a month old?
    if ka.na.Timestamp.Before(time.Now().Add(-1 * numMissingDays * time.Hour * 24)) {
        return true
    }

    // Never succeeded?
    if ka.lastsuccess.IsZero() && ka.attempts >= numRetries {
        return true
    }

    // Hasn't succeeded in too long?
    if !ka.lastsuccess.After(time.Now().Add(-1*minBadDays*time.Hour*24)) &&
        ka.attempts >= maxFailures {
        return true
    }

    return false
}
```

至此，地址管理服务相关地址维护功能基本看过。但是上面的逻辑，是从第一次没有节点情况，从dns得到的节点。当此节点连接到其它节点之后，会发一个获取地址的消息，让其它节点给一份它自己的地址信息。

## 1.3. 获取随机地址方法

当地址管理器中有地址之后，上层服务就要调用addrmanager得到地址去连结。因此，addrmanager会有一个很重要的方法GetAddress。

### 1.3.1. GetAddress

```go
// GetAddress returns a single address that should be routable.  It picks a
// random one from the possible addresses with preference given to ones that
// have not been used recently and should not pick 'close' addresses
// consecutively.
func (a *AddrManager) GetAddress() *KnownAddress {
    // Protect concurrent access.
    a.mtx.Lock()
    defer a.mtx.Unlock()

    if a.numAddresses() == 0 {
        return nil
    }

    // Use a 50% chance for choosing between tried and new table entries.
    if a.nTried > 0 && (a.nNew == 0 || a.rand.Intn(2) == 0) {
        // Tried entry.
        large := 1 << 30
        factor := 1.0
        for {
            // pick a random bucket.
            bucket := a.rand.Intn(len(a.addrTried))
            if a.addrTried[bucket].Len() == 0 {
                continue
            }

            // Pick a random entry in the list
            e := a.addrTried[bucket].Front()
            for i :=
                a.rand.Int63n(int64(a.addrTried[bucket].Len())); i > 0; i-- {
                e = e.Next()
            }
            ka := e.Value.(*KnownAddress)
            randval := a.rand.Intn(large)
            if float64(randval) < (factor * ka.chance() * float64(large)) {
                log.Tracef("Selected %v from tried bucket",
                    NetAddressKey(ka.na))
                return ka
            }
            factor *= 1.2
        }
    } else {
        // new node.
        // XXX use a closure/function to avoid repeating this.
        large := 1 << 30
        factor := 1.0
        for {
            // Pick a random bucket.
            bucket := a.rand.Intn(len(a.addrNew))
            if len(a.addrNew[bucket]) == 0 {
                continue
            }
            // Then, a random entry in it.
            var ka *KnownAddress
            nth := a.rand.Intn(len(a.addrNew[bucket]))
            for _, value := range a.addrNew[bucket] {
                if nth == 0 {
                    ka = value
                }
                nth--
            }
            randval := a.rand.Intn(large)
            if float64(randval) < (factor * ka.chance() * float64(large)) {
                log.Tracef("Selected %v from new bucket",
                    NetAddressKey(ka.na))
                return ka
            }
            factor *= 1.2
        }
    }
}
```

>GetAddress 返回一个**随机**的地址给上层。

- 在已经连结过的节点(addrTried)和新节点(addrNew)之间选择概率为50%
- 随机选择一个bucket
- 在bucket中随机选择一个项
- 随机因子加机会权重
  
**主要看下因子逻辑:**

factor随着失败的次数变大而变大，因此成功的概率也在变大。进入ka.chance()看下。地址的权重是如何处理的：

```go
// chance returns the selection probability for a known address.  The priority
// depends upon how recently the address has been seen, how recently it was last
// attempted and how often attempts to connect to it have failed.
func (ka *KnownAddress) chance() float64 {
    now := time.Now()
    lastAttempt := now.Sub(ka.lastattempt)

    if lastAttempt < 0 {
        lastAttempt = 0
    }

    c := 1.0

    // Very recent attempts are less likely to be retried.
    if lastAttempt < 10*time.Minute {
        c *= 0.01
    }

    // Failed attempts deprioritise.
    for i := ka.attempts; i > 0; i-- {
        c /= 1.5
    }

    return c
}
```

可以看出，最近10分钟内使用过的，权重会变很小。失败的次数越多，权重也是越小。


## 1.4. 从连接的节点获取

### 1.4.1. 获取节点地址

我们先不管消息的发送，只看收到其它节点发送回来的地址的代码。节点之前沟通的消息处理都是在server.go中。因此，我们在这个文件中找到了OnAddr。来看看

```go
// OnAddr is invoked when a peer receives an addr bitcoin message and is
// used to notify the server about advertised addresses.
func (sp *serverPeer) OnAddr(_ *peer.Peer, msg *wire.MsgAddr) {
    // Ignore addresses when running on the simulation test network.  This
    // helps prevent the network from becoming another public test network
    // since it will not be able to learn about other peers that have not
    // specifically been provided.
    if cfg.SimNet {
        return
    }

    // Ignore old style addresses which don't include a timestamp.
    if sp.ProtocolVersion() < wire.NetAddressTimeVersion {
        return
    }

    // A message that has no addresses is invalid.
    if len(msg.AddrList) == 0 {
        peerLog.Errorf("Command [%s] from %s does not contain any addresses",
            msg.Command(), sp.Peer)
        sp.Disconnect()
        return
    }

    for _, na := range msg.AddrList {
        // Don't add more address if we're disconnecting.
        if !sp.Connected() {
            return
        }

        // Set the timestamp to 5 days ago if it's more than 24 hours
        // in the future so this address is one of the first to be
        // removed when space is needed.
        now := time.Now()
        if na.Timestamp.After(now.Add(time.Minute * 10)) {
            na.Timestamp = now.Add(-1 * time.Hour * 24 * 5)
        }

        // Add address to known addresses for this peer.
        sp.addKnownAddresses([]*wire.NetAddress{na})
    }

    // Add addresses to server address manager.  The address manager handles
    // the details of things such as preventing duplicate addresses, max
    // addresses, and last seen updates.
    // XXX bitcoind gives a 2 hour time penalty here, do we want to do the
    // same?
    sp.server.addrManager.AddAddresses(msg.AddrList, sp.NA())
}
```

它收到地址列表之后，会调用addrManager.AddAddresses添加到本地。同时这个
sp.NA() 返回此peer的地址信息，作为这批地址的源。

### 1.4.2. 把地址给其它节点

当其它节点请求地址列表时，此节点会从自己的缓存中取出地址返回。

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

AddressCache()返回自己节点23%的地址。并且是随机生成。然后调用sp.pushAddrMsg(addrCache)发送出去。

```go
// AddressCache returns the current address cache.  It must be treated as
// read-only (but since it is a copy now, this is not as dangerous).
func (a *AddrManager) AddressCache() []*wire.NetAddress {
    a.mtx.Lock()
    defer a.mtx.Unlock()

    addrIndexLen := len(a.addrIndex)
    if addrIndexLen == 0 {
        return nil
    }

    allAddr := make([]*wire.NetAddress, 0, addrIndexLen)
    // Iteration order is undefined here, but we randomise it anyway.
    for _, v := range a.addrIndex {
        allAddr = append(allAddr, v.na)
    }

    numAddresses := addrIndexLen * getAddrPercent / 100
    if numAddresses > getAddrMax {
        numAddresses = getAddrMax
    }

    // Fisher-Yates shuffle the array. We only need to do the first
    // `numAddresses' since we are throwing the rest.
    for i := 0; i < numAddresses; i++ {
        // pick a number between current index and the end
        j := rand.Intn(addrIndexLen-i) + i
        allAddr[i], allAddr[j] = allAddr[j], allAddr[i]
    }

    // slice off the limit we are willing to share.
    return allAddr[0:numAddresses]
}
```